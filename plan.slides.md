# План

## Тестирование

---

### Проверка вывода

#### Железо

#### Виртуальная машина

---

### Единый сценарий

---

#### Сценарий простейший

```lua
require "fixture"
require "lib"

function build()
    print "build"
    assert(
        os.execute("...")
    )

    return string.format("%s/build/qemu-image", global_workspace_dir)
end

function test()
    print "test"
    output:expect_line("Hello world!", 2)
    pass()
end
```

---

#### Сценарий с сервером (1)

```lua
require "lib"

function unreliable_setup()
    print "unreliable_setup"

    output_with_tag("CHECK", "Checking if log contains boot message...")
    output:wait_for_line("[BOOT] Starting ...", 20)
end

function setup()
    print "setup"

    server = Process.new(string.format("%s/build/host/server/server", global_workspace_dir))
    server.run()
end

function teardown()
    print "teardown"

    server = nil
    collectgarbage()
end
```
---

#### Сценарий с сервером (2)

```lua
function test()
    print "test"
    server_output = server:get_output()

    output:expect_line("Client sent: perform read write operations ...", 5)
    server_output:expect_line("Server received: perform read write operations ...", 1)
    pass()
end
```

---

#### Конфигурация машины

---

##### Qemu

```lua
MACHINE_CONFIG = {
    type = "qemu",
    inner = {
        command_line = "/opt/toolchain/bin/qemu-system-x86_64 -m 1024 -cpu core2duo -serial stdio -kernel {kernel_image}"
    }
}
```

---

##### Физическая

```lua
MACHINE_CONFIG = {
    type = "hw",
    inner = {
        serial_port_name = string.format("tests/scenarios/%s/reference_output", global_test_name),
        -- serial_port_name = "/dev/ttyUSB0",
        serial_port_config = "0:4:18b2:a30:3:1c:7f:15:4:0:1:0:11:13:1a:0:12:f:17:16:0:0:0:0:0:0:0:0:0:0:0:0:0:0:0:0",
        power_management = "manual"
        -- pdu_host_name = "pdu.example.com"
        -- outlet_number = 6
    }
}
```

---

### Надёжное управление ресурсами

#### Жизненный цикл машин

#### Жизненный цикл процессов

---

## Lua

### Типы данных

### Прототипное ОО

---

### Особые поля

#### __index

#### __gc

---

### Корутины

---

### FFI

#### Стек

#### Реестр

---

## Библиотеки

У всех чуть разный API, что создаёт кучу проблем при попытке перейти на другую
библиотеку.

---

### hlua

- высокоуровневая, работает через стек

- 5.2

- довольно ограниченная, много багов

- `Lua: Send`

- я сюда контрибьютил (полгода назад)

---

### td_rlua

- высокоуровневая, работает через стек

- 5.3

- форк hlua, слабо развивается

---

### rlua

- высокоуровневая, работает через реестр

- 5.3

- новый проект, пока активно развивается, выглядит разумно

- `Lua: ! Send`

---

## Стек hlua

- hlua. Высокоуровневая обёртка.
- lua52-sys. Непосредственно FFI.
- lua (компилируется build.rs). Интерпретатор как программа на C.

---

### hlua

```rust
impl<'lua, L> LuaRead<L> for HashMap<AnyHashableLuaValue, AnyLuaValue>
    where L: AsMutLua<'lua>
{
    // TODO: this should be implemented using the LuaTable API instead of raw Lua calls.
    fn lua_read_at_position(lua: L, index: i32) -> Result<Self, L> {
        let mut me = lua;
        unsafe { ffi::lua_pushnil(me.as_mut_lua().0) };
        let index = index - 1;
        let mut result = HashMap::new();

        // { next slide }

        Ok(result)
    }
}

```

---

```rust
        loop {
            if unsafe { ffi::lua_next(me.as_mut_lua().0, index) } == 0 {
                break;
            }

            let key = {
                let maybe_key: Option<AnyHashableLuaValue> =
                    LuaRead::lua_read_at_position(&mut me, -2).ok();
                match maybe_key {
                    None => {
                        // Cleaning up after ourselves
                        unsafe { ffi::lua_pop(me.as_mut_lua().0, 2) };
                        return Err(me)
                    }
                    Some(k) => k,
                }
            };

            let value: AnyLuaValue =
                LuaRead::lua_read_at_position(&mut me, -1).ok().unwrap();

            unsafe { ffi::lua_pop(me.as_mut_lua().0, 1) };

            result.insert(key, value);
        }
```

---

### lua52-sys

```rust
#[inline(always)]
pub fn lua_upvalueindex(i: c_int) -> c_int {
    LUA_REGISTRYINDEX - i
}

#[inline(always)]
pub unsafe fn lua_call(L: *mut lua_State, nargs: c_int, nresults: c_int) {
    lua_callk(L, nargs, nresults, 0, None)
}

#[inline(always)]
pub unsafe fn lua_pcall(L: *mut lua_State, nargs: c_int, nresults: c_int, errfunc: c_int) -> c_int {
    lua_pcallk(L, nargs, nresults, errfunc, 0, None)
}

#[inline(always)]
pub unsafe fn lua_yield(L: *mut lua_State, nresults: c_int) -> c_int {
    lua_yieldk(L, nresults, 0, None)
}

#[inline(always)]
pub unsafe fn lua_pop(L: *mut lua_State, n: c_int) {
    lua_settop(L, -n-1)
}
```

---

## Что может hlua

https://github.com/tomaka/hlua#reading-and-writing-variables

---

### Чтение и запись переменных

### Выполнение кода на Lua

### Оборачивание функций на Rust в Lua

#### Прокидывание Result

Вызов `error()` в Lua.

---

### Работа с таблицами

### Вызов функций Lua из Rust

### Чтение и запись контейнеров

### Пользовательские данные (userdata)

---

## Как транслируются объекты

---

### Деструктор

Деструктор записывается в `__gc`. Если сделать `a = nil; collectgarbage()` это
приведёт к его вызову.

---

### Методы

Методы записываются в метатаблицу. В простейшем случае так:

```rust
struct Foo;

impl<L> hlua::Push<L> for Foo where L: hlua::AsMutLua<'lua> {
    fn push_to_lua(self, lua: L) -> hlua::PushGuard<L> {
        lua::userdata::push_userdata(self, lua,
            |mut metatable| {
                // you can define all the member functions of Foo here
                // see the official Lua documentation for metatables
                metatable.set("__call", hlua::function0(|| println!("hello from foo")))
            })
    }
}

fn main() {
    let mut lua = lua::Lua::new();
    lua.set("foo", Foo);
    lua.execute::<()>("foo()");       // prints "hello from foo"
}
```

---

## Реализуем Lua API для виртуальной и настоящей машины

---

### Машины

```rust
fn init_machines<'a>(
    machine: &mut hlua::LuaTable<
        hlua::PushGuard<&mut hlua::LuaTable<hlua::PushGuard<&mut hlua::Lua<'a>>>>,
    >,
    machines: Arc<Mutex<HashMap<usize, Machine>>>,
) {
    let new = {
        // ...
    };
    let reset_and_run = {
        let machines = machines.clone();
        move |machine_lua: &mut MachineLua, image_path: String| {
            let handle = machine_lua.handle();
            let mut machines = machines.lock().unwrap();
            let machine = &mut machines.get_mut(&handle).unwrap();
            machine.reset_and_run(image_path)
        }
    };
    let drop = {
        let machines = machines.clone();
        move |machine_lua: &mut MachineLua| {
            let handle = machine_lua.handle();
            let _ = machines.lock().unwrap().remove(&handle);
        }
    };

    machine.set("new", hlua::function2(new));
    // ...
}
```

---

### Machine::new

```rust
impl Machine {
    pub fn new(
        config: HashMap<hlua::AnyHashableLuaValue, hlua::AnyLuaValue>,
        config_inner: HashMap<hlua::AnyHashableLuaValue, hlua::AnyLuaValue>,
    ) -> Machine {
        let ty = &config[&hlua::AnyHashableLuaValue::LuaString("type".to_owned())];
        match ty {
            &hlua::AnyLuaValue::LuaString(ref s) => match s.as_ref() {
                "hw" => {
                    // ...

                    Machine {
                        internals: Internals::Hw(HwInternals {
                            serial_port_name: serial_port_name,
                            serial_port_settings: serial_port_config,
                            serial_port: None,
                            power_management: power_management_config,
                        }),
                    }
                }
                "qemu" => {
                    // ...

                    Machine {
                        internals: Internals::Qemu(QemuInternals {
                            command_line: command_line.clone(),
                            state: None,
                        }),
                    }
                }
            },
        }
    }
```

---

### Qemu

```rust
pub struct Qemu {
    process: Child,
    output_queue: mpsc::Receiver<String>,
    _reader_thread: JoinHandle<()>,
}

impl Drop for Qemu {
    fn drop(&mut self) {
        self.process.kill().unwrap();
    }
}
```

---

```rust
impl Qemu {
    pub fn new(command: &str) -> Self {
        // next slide
    }
    pub fn read_line(&mut self) -> Option<String> {
        let result = self.output_queue.try_recv();
        match result {
            Ok(s) => Some(s.trim().to_owned()),
            Err(_) => None,
        }
    }
}
```

---

```rust
        let args: Vec<_> = command.split(' ').collect();
        let command = args[0];
        let mut to_be_child = Command::new(command);

        to_be_child.stdin(Stdio::piped()).stdout(Stdio::piped());

        to_be_child.args(args[1..].iter());

        let mut child = to_be_child.spawn().expect("failed to execute child");
        info!("Qemu PID: {}", child.id());

        let stdout = child.stdout.take().unwrap();
        let mut stdout = BufReader::new(stdout);

        let (output_queue_tx, output_queue_rx) = mpsc::channel();

        let thread = thread::spawn({
            move || loop {
                let mut line = String::new();
                stdout.read_line(&mut line).unwrap();
                {
                    output_queue_tx.send(line).unwrap();
                }
            }
        });

        Qemu {
            process: child,
            _reader_thread: thread,
            output_queue: output_queue_rx,
        }
```

---

```rust
pub struct QemuLua {
    handle: usize,
}

impl QemuLua {
    pub fn new(handle: usize) -> Self {
        Self { handle }
    }
    pub fn handle(&self) -> usize {
        self.handle
    }
}

implement_lua_push!(QemuLua, |_| {});
implement_lua_read!(QemuLua);
```

---

### machine.lua

```lua
inspect = require("inspect")

Machine = {config = nil, output = nil, status = nil, inner = nil}

function Machine.new(config)
    local inner = core.machine.new(config, config.inner)
    local self = {config = config, output = nil, status = "unknown", inner = inner}

    local reset_and_run = function(image_path)
        core.machine.reset_and_run(self.inner, image_path)
        return self.inner
    end

    local get_output =
        function()
        -- next slide
    end

    local power_down = function()
        core.machine.power_down(self.inner)
    end

    return {
        reset_and_run = reset_and_run,
        get_output = get_output,
        power_down = power_down
    }
end
```

---

```lua
        if not self.output then
            self.output = {}
            self.output.mt = {
                __index = {
                    read_line = function()
                        return core.machine.read_line(self.inner)
                    end,
                    wait_for_line = function(self, l, t, s)
                        return wait_for_line(self, l, t, s)
                    end,
                    expect_line = function(self, l, t, s)
                        return expect_line(self, l, t, s)
                    end,
                    assert_line = function(self, l, t, s)
                        return assert_line(self, l, t, s)
                    end
                }
            }
            setmetatable(self.output, self.output.mt)
        end
        return self.output
```

---

### `wait_for_line`

```lua
function wait_for_line(source, expected_line, timeout, time_step)
    local status, err, ret = xpcall(wait_for_line_panicking, debug.traceback, source, expected_line, timeout, time_step)

    if not status then
        print(err)
        global_result = "fail"
    end

    return ret
end
```

---

### `wait_for_line_panicking`

```lua
function wait_for_line_panicking(source, expected_line, timeout, time_step)
    time_step = time_step or 0.1
    local result = "timeout"
    local spent_time = 0

    while spent_time < timeout do
        local line = source:read_line()
        if line == expected_line then
            result = "received"
            break
        end
        if not line or line:len() == 0 then
            core.sleep(0, time_step * 1000)
            spent_time = spent_time + time_step
        end
    end

    assert(result ~= "timeout", string.format("did not get '%s' in %d seconds", expected_line, timeout))
end
```

---

## tokio-process & tokio-serial

```rust
impl AsyncRead for Serial {
    unsafe fn prepare_uninitialized_buffer(&self, buf: &mut [u8]) -> bool;
    fn read_buf<B>(&mut self, buf: &mut B) -> Result<Async<usize>, Error> where
        B: BufMut, ;
    fn framed<T>(self, codec: T) -> Framed<Self, T> where
        Self: AsyncWrite,
        T: Decoder + Encoder, ;
    fn split(self) -> (ReadHalf<Self>, WriteHalf<Self>) where
        Self: AsyncWrite, ;
}

impl AsyncWrite for Serial {
    fn shutdown(&mut self) -> Poll<(), Error>;
    fn write_buf<B>(&mut self, buf: &mut B) -> Result<Async<usize>, Error> where
        B: Buf, ;
}
```

---

### tokio-process

#### Примеры

---

##### Запуск процесса с асинхронным IO

```rust
extern crate futures;
extern crate tokio_core;
extern crate tokio_process;

use std::process::Command;

use futures::Future;
use tokio_core::reactor::Core;
use tokio_process::CommandExt;

fn main() {
    // Create our own local event loop
    let mut core = Core::new().unwrap();

    // Use the standard library's `Command` type to build a process and
    // then execute it via the `CommandExt` trait.
    let child = Command::new("echo").arg("hello").arg("world")
                        .spawn_async(&core.handle());

    // Make sure our child succeeded in spawning
    let child = child.expect("failed to spawn");

    match core.run(child) {
        Ok(status) => println!("exit status: {}", status),
        Err(e) => panic!("failed to wait for exit: {}", e),
    }
}
```

---

##### Захват вывода

```rust
extern crate futures;
extern crate tokio_core;
extern crate tokio_process;

use std::process::Command;

use futures::Future;
use tokio_core::reactor::Core;
use tokio_process::CommandExt;

fn main() {
    let mut core = Core::new().unwrap();

    // Like above, but use `output_async` which returns a future instead of
    // immediately returning the `Child`.
    let output = Command::new("echo").arg("hello").arg("world")
                        .output_async(&core.handle());
    let output = core.run(output).expect("failed to collect output");

    assert!(output.status.success());
    assert_eq!(output.stdout, b"hello world\n");
}
```

---

##### Чтение по строкам

```rust
extern crate futures;
extern crate tokio_core;
extern crate tokio_process;
extern crate tokio_io;

use std::io;
use std::process::{Command, Stdio, ExitStatus};

use futures::{BoxFuture, Future, Stream};
use tokio_core::reactor::Core;
use tokio_process::{CommandExt, Child};

fn print_lines(mut cat: Child) -> BoxFuture<ExitStatus, io::Error> {
    let stdout = cat.stdout().take().unwrap();
    let reader = io::BufReader::new(stdout);
    let lines = tokio_io::io::lines(reader);
    let cycle = lines.for_each(|l| {
        println!("Line: {}", l);
        Ok(())
    });
    cycle.join(cat).map(|((), s)| s).boxed()
}

fn main() {
    let mut core = Core::new().unwrap();
    let mut cmd = Command::new("cat");
    let mut cat = cmd.stdout(Stdio::piped());
    let child = cat.spawn_async(&core.handle()).unwrap();
    core.run(print_lines(child)).unwrap();
}
```

---

#### Подводные грабли

```
While similar to the standard library, this crate's Child type differs
importantly in the behavior of drop. In the standard library, a child process
will continue running after the instance of std::process::Child is dropped. In
this crate, however, because tokio_process::Child is a future of the child's
ExitStatus, a child process is terminated if tokio_process::Child is dropped.
The behavior of the standard library can be regained with the Child::forget
method.
```

---

### Грабли Tokio в мире Lua

- Event loop Tokio существует только на стороне Rust. Lua не имеет к нему доступа.
- Физически поток управления уходит в Lua, и оттуда он не выйдет по событиям поллинга Tokio.
- Нужна виртуальная машина Lua с поддержкой Tokio
- ???
- NodeLua на Rust
- PROFIT!

---

## Backup

---

### Метаметоды Lua

- add
- sub
- mul
- div
- mod
- pow
- unm
- concat
- len
- eq
- lt
- le
- index
- newindex
- call

---

### Зачем нужны NLL

```rust
    pub fn reset_and_run(&mut self, image_path: String) {
        // This match is here to not borrow self.hw_int.power_management
        // in following match because otherwise we can't call method
        // borrowing &mut self
        let need_delay = match self.internals {
            Internals::Qemu(_) => false,
            Internals::Hw(ref hw_int) => {
                if let PowerManagement::Auto(_, _) = hw_int.power_management {
                    true
                } else {
                    false
                }
            }
        };
        match self.internals {
            Internals::Qemu(_) => {}
            Internals::Hw(_) => {
                info!("Resetting HW");
                {
                    self.power_down();
                }
                if need_delay {
                    thread::sleep(Duration::from_secs(10));
                }
                {
                    self.power_up();
                }
            }
        }
        self.run(image_path)
    }
```

---
