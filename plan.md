# План

## Тестирование

### Проверка вывода

#### Железо

#### Виртуальная машина

### Единый сценарий

## Lua

### Типы данных

### Прототипное ОО

### Особые поля

#### __index

#### __gc

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