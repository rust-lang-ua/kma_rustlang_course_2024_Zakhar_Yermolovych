### Попередні налаштування

Впевніться що у вас встановлені необхідні складові:

- Python 3.7 і вище (CPython, PyPy, та GraalPy)
- Rust 1.63 і вище

---

### Використання Rust з Python

PyO3 можна використовувати для створення нативного модуля Python. Найпростіший спосіб спробувати це вперше - скористатися maturin. maturin - це інструмент для створення та публікації пакунків Python на основі Rust з мінімальною конфігурацією. На наступних кроках ви встановите maturin, згенеруєте та зберете новий пакунок Python, а потім запустите Python, щоб імпортувати та виконати функцію з пакунка.

---

### Крок 1

Спочатку створіть нову директорію з новим віртуальним каталогом Python virtualenv

```bash
mkdir password_hasher
cd password_hasher
python -m venv .env
```

та

```bash
source .env/bin/activate
```

---

Якщо ви використовуєте PowerShell, команда буде такою:

```bash
.\.env\Scripts\Activate

```

---

Якщо ви використовуєте Command Prompt (cmd):

```bash
.env\Scripts\activate
```

**Переконатися, що `activate` існує**:

- Файл `activate` знаходиться у папці `bin` (на Linux/macOS) або `Scripts` (на Windows) всередині вашого віртуального середовища.

Команда `source .env/bin/activate` використовується для активації **віртуального середовища Python**, яке створюється за допомогою інструмента `venv` (або інших, наприклад, `virtualenv`). Віртуальне середовище дозволяє ізолювати пакети Python для конкретного проєкту, уникнувши конфліктів між залежностями різних проєктів.

### Для чого це потрібно?

1. **Ізоляція проєкту**: Використовуються тільки ті бібліотеки, які встановлені у вашому `.env`.
2. **Легка передача**: Ви можете передати файл `requirements.txt` (перелік залежностей) іншому розробнику, і вони зможуть створити таке ж середовище.
3. **Уникнення конфліктів**: Наприклад, проєкт A використовує Flask 2.x, а проєкт B — Flask 1.x. У віртуальному середовищі ці версії не конфліктуватимуть.

---

### Як деактивувати середовище?

Для виходу з віртуального середовища введіть:

```bash
deactivate
```

Це повертає термінал до стандартного стану, і Python/Pip знову використовуватимуть глобальні налаштування.

---

### Що відбувається після виконання команди?

1. **Активація віртуального середовища**:

   - `source` — це команда для виконання скриптів у поточному термінальному сеансі.
   - `.env/bin/activate` — це скрипт, який:
     - Змінює значення змінної оточення `PATH`, щоб Python і `pip` використовувалися з віртуального середовища.
     - Встановлює змінну `VIRTUAL_ENV`, яка вказує на активне віртуальне середовище.

2. **Зміна Python і pip**:

   - Після активації всі команди `python` та `pip` посилаються на версії, встановлені у віртуальному середовищі (`.env`), а не глобально встановлені.

3. **Зміна вигляду командного рядка**:
   - Багато середовищ додають назву віртуального середовища перед запрошенням командного рядка, наприклад:
     ```
     (.env) user@machine:~/project$
     ```

---

### Крок 2: Встановленя maturin у віртуальний каталог

```bash
pip install maturin
```

### Крок 3: Створеня проекту

Все ще перебуваючи у цьому каталозі `password_hasher`, виконайте `maturin init`. Це призведе до створення нового коду пакунка. Коли вам буде запропоновано вибір прив'язок для використання, виберіть прив'язки pyo3:

```bash
$ maturin init
✔ 🤷 What kind of bindings to use? · pyo3
  ✨ Done! New project created password_hasher
```

Найважливішими файлами, які генеруються цією командою, є `Cargo.toml` і `lib.rs`, які виглядатимуть приблизно так, як показано нижче:

**`Cargo.toml`**

```toml
[package]
name = "password_hasher"
version = "0.1.0"
edition = "2021"

[lib]
# Назва рідної бібліотеки. Це ім'я, яке буде використано у Python для імпорту
# бібліотеки (тобто `import password_hasher`). Якщо ви змінюєте його, ви також маєте змінити назву
# `#[pymodule]` у `rc/lib.rs`.
name = "password_hasher"
# "cdylib" необхідний для створення спільної бібліотеки, з якої Python може імпортувати дані.
#
# Подальший код Rust (включно з кодом у `bin/`, `examples/` та `tests/`) не зможе
# використовувати password_hasher;` якщо не включено тип ящика "rlib" або "lib", наприклад:
# crate-type = ["cdylib", "rlib"]
crate-type = ["cdylib"]

[dependencies]
pyo3 = { version = "0.22.5", features = ["extension-module"] }
```

---

### Крок 4: Додавання залежностей

Для нашого проекту треба додати деякі залежності.

```toml
[package]
name = "password_hasher"
version = "0.1.0"
edition = "2021"


[dependencies]
pyo3 = { version = "0.22.6", features = ["extension-module"] }
argon2 = "0.5.3"
rand = "0.8.5"

[lib]
crate-type = ["cdylib"]
```

---

### Крок 5: Хешування паролю

Пишемо просту реалізацію хешування паролю і його верифікацію

```rust
use argon2::{
    password_hash::{rand_core::OsRng, PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};
use pyo3::prelude::*;

#[pyfunction]
fn hash_password(password: &str) -> PyResult<String> {
    let salt = SaltString::generate(&mut OsRng);
    // Argon2 з параметрами за замовчуванням (Argon2id v19)
    let argon2 = Argon2::default();
    // Хеш-пароль до рядка PHC ($argon2id$v=19$...)
    let password_hash = argon2
        .hash_password(password.as_bytes(), &salt)
        .map_err(|e| pyo3::exceptions::PyValueError::new_err(e.to_string()))?
        .to_string();
    Ok(password_hash)
}

#[pyfunction]
fn verify_password(password: &str, password_hash: &str) -> PyResult<bool> {
    // Звірити пароль з PHC-рядком.
    let parsed_hash = PasswordHash::new(password_hash)
        .map_err(|e| pyo3::exceptions::PyValueError::new_err(e.to_string()))?;
    let is_valid = Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok();

    Ok(is_valid)
}
/// Python модуль на основі Rust
#[pymodule]
fn password_hasher(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(hash_password, m)?)?;
    m.add_function(wrap_pyfunction!(verify_password, m)?)?;
    Ok(())
}

```

---

#### Крок 6: Збірка модуля з `maturin`

1. Зберіть і встановіть модуль у локальному середовищі Python:

```bash
maturin develop
```

Ця команда створить і встановить модуль у Python.

2. Перевірте, чи все працює:

Ось як створити файл Python і запустити його після виконання `maturin develop`:

#### Кроки:

1. **Створіть Python-файл**:

   - У кореневій папці вашого проекту (де знаходиться `Cargo.toml`) створіть файл, наприклад, `test_password_hasher.py`.
   - Збережіть у нього код:

   ```python
   import password_hasher

   # Хешування паролю
   hashed = password_hasher.hash_password("my_secure_password")
   print("Hashed password:", hashed)

   # Перевірка паролю
   is_valid = password_hasher.verify_password("my_secure_password", hashed)
   print("Is valid:", is_valid)
   ```

2. **Запустіть файл**:

   - Відкрийте термінал у тому ж каталозі, де розташований `test_password_hasher.py`.
   - Виконайте команду:

   ```bash
   python test_password_hasher.py
   ```

3. **Перевірте результат**:
   - У терміналі ви побачите щось подібне:
   ```
   Hashed password: $argon2id$v=19$m=19456,t=2,p=1$...
   Is valid: True
   ```

---

#### Крок 7: Збірка модуля для розповсюдження

Якщо ви хочете створити розповсюджуваний модуль (наприклад, `.whl` файл), виконайте:

```bash
maturin build --release
```

Це створить файл колеса (`.whl`) у папці `target/wheels`, який можна розповсюджувати або встановлювати в інших середовищах Python.

---

### Підсумок

1. Ми створили проект на Rust з `pyo3` і `argon2`.
2. Реалізували функції `hash_password` та `verify_password`.
3. Використали `maturin` для збирання та встановлення модуля в Python.

Тепер ви маєте Python-модуль, написаний на Rust, що підтримує хешування паролів!
