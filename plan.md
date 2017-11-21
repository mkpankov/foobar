# План

## Тестирование

### Проверка вывода

#### Железо

#### Виртуальная машина

### Единый сценарий

### Надёжное управление ресурсами

#### Жизненный цикл машин

#### Жизненный цикл процессов

## Lua

### Типы данных

### Прототипное ОО

### Особые поля

#### __index

#### __gc

### Корутины

### FFI

#### Стек

#### Реестр

## Библиотеки

У всех чуть разный API, что создаёт кучу проблем при попытке перейти на другую
библиотеку.

### hlua

Высокоуровневая, работает через стек. 5.2

Довольно ограниченная, много багов.

Lua: Send.

Я сюда контрибьютил (полгода назад).

### td_rlua

Высокоуровневая, работает через стек. 5.3

Форк hlua, слабо развивается.

### rlua

Высокоуровневая, работает через реестр. 5.3

Новый проект, пока активно развивается.

Lua: не Send!

## Стек hlua

- hlua. Высокоуровневая обёртка.
- lua52-sys. Непосредственно FFI.
- lua (компилируется build.rs). Интерпретатор как программа на C.

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

        Ok(result)
    }
}

```

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

#[inline(always)]
pub unsafe fn lua_newtable(L: *mut lua_State) {
    lua_createtable(L, 0, 0)
}
```

## Что может hlua

### Чтение и запись переменных

### Выполнение кода на Lua

### Оборачивание функций на Rust в Lua

#### Прокидывание Result

Вызов `error()` в Lua.

### Работа с таблицами

### Вызов функций Lua из Rust

### Чтение и запись контейнеров

### Пользовательские данные (userdata)

## Как транслируются объекты

Деструктор записывается в `__gc`. Если сделать `a = nil; collectgarbage()` это
приведёт к его вызову.

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

## Реализуем Lua API для виртуальной и настоящей машины

### Машины

```rust
fn init_machines<'a>(
    machine: &mut hlua::LuaTable<
        hlua::PushGuard<&mut hlua::LuaTable<hlua::PushGuard<&mut hlua::Lua<'a>>>>,
    >,
    machines: Arc<Mutex<HashMap<usize, Machine>>>,
) {
    let new = {
        let machines = machines.clone();
        move |config: HashMap<_, _>, config_inner: HashMap<_, _>| {
            let machine = Machine::new(config, config_inner);
            let handle = machine.new_handle(&mut *machines.lock().unwrap());
            MachineLua::new(handle)
        }
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
    let power_down = {
        let machines = machines.clone();
        move |machine_lua: &mut MachineLua| {
            let handle = machine_lua.handle();
            let mut machines = machines.lock().unwrap();
            let machine = &mut machines.get_mut(&handle).unwrap();
            machine.power_down()
        }
    };
    let read_line = {
        let machines = machines.clone();
        move |machine_lua: &mut MachineLua| {
            let handle = machine_lua.handle();
            let mut machines = machines.lock().unwrap();
            let machine = machines.get_mut(&handle).unwrap();
            machine.read_line()
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
    machine.set("reset_and_run", hlua::function2(reset_and_run));
    machine.set("power_down", hlua::function1(power_down));
    machine.set("read_line", hlua::function1(read_line));
    machine.set("drop", hlua::function1(drop));
}
```

### Machine::new

```rust
impl Machine {
    pub fn new(
        config: HashMap<hlua::AnyHashableLuaValue, hlua::AnyLuaValue>,
        config_inner: HashMap<hlua::AnyHashableLuaValue, hlua::AnyLuaValue>,
    ) -> Machine {
        info!("config: {:?}", config);
        info!("config_inner: {:?}", config_inner);

        let ty = &config[&hlua::AnyHashableLuaValue::LuaString("type".to_owned())];
        match ty {
            &hlua::AnyLuaValue::LuaString(ref s) => match s.as_ref() {
                "hw" => {
                    let serial_port_name = &config_inner
                        [&hlua::AnyHashableLuaValue::LuaString("serial_port_name".to_owned())];
                    let serial_port_name = match serial_port_name {
                        &hlua::AnyLuaValue::LuaString(ref s) => s.clone(),
                        _ => panic!("No serial port name for Hw machine config"),
                    };
                    let serial_port_config = &config_inner
                        [&hlua::AnyHashableLuaValue::LuaString("serial_port_config".to_owned())];
                    let serial_port_config = match serial_port_config {
                        &hlua::AnyLuaValue::LuaString(ref s) => s.clone(),
                        _ => panic!("No serial port config for Hw machine config"),
                    };

                    let power_management = &config_inner
                        [&hlua::AnyHashableLuaValue::LuaString("power_management".to_owned())];
                    let power_management = match power_management {
                        &hlua::AnyLuaValue::LuaString(ref s) => s.clone(),
                        _ => panic!("No power management setting for Hw machine config"),
                    };

                    let power_management_config = if power_management == "auto" {
                        let t = &config_inner
                            [&hlua::AnyHashableLuaValue::LuaString("pdu_host_name".to_owned())];
                        let pdu_host_name = match t {
                            &hlua::AnyLuaValue::LuaString(ref s) => s.clone(),
                            _ => panic!("No power management setting for Hw machine config"),
                        };

                        let t = &config_inner
                            [&hlua::AnyHashableLuaValue::LuaString("outlet_number".to_owned())];
                        let outlet_number = match t {
                            &hlua::AnyLuaValue::LuaNumber(n) => n,
                            _ => panic!("No power management setting for Hw machine config"),
                        };
                        PowerManagement::Auto(
                            PduHostName(pdu_host_name),
                            OutletNumber(outlet_number as u8),
                        )
                    } else {
                        PowerManagement::Manual
                    };

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
                    let command_line = &config_inner
                        [&hlua::AnyHashableLuaValue::LuaString("command_line".to_owned())];
                    let command_line = match command_line {
                        &hlua::AnyLuaValue::LuaString(ref s) => s,
                        _ => panic!("No command line for Qemu machine config"),
                    };
                    Machine {
                        internals: Internals::Qemu(QemuInternals {
                            command_line: command_line.clone(),
                            state: None,
                        }),
                    }
                }
                _ => unreachable!(),
            },
            _ => unreachable!(),
        }
    }
```

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

impl Qemu {
    pub fn new(command: &str) -> Self {
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
    }
    pub fn read_line(&mut self) -> Option<String> {
        let result = self.output_queue.try_recv();
        match result {
            Ok(s) => Some(s.trim().to_owned()),
            Err(_) => None,
        }
    }
}

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

## Backup

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