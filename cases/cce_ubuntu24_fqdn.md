### Q:
В облаке Google ВМ с Ubuntu 24 на $(hostname) даёт FQDN, а не короткое имя.
А ЕТСД глуп в этом плане.

### A1:
Проверил еще раз. Ubuntu возвращает и там, и там точно то, что было назначено в sudo hostnamectl set-hostname XXX. Если был указан fqdn - вернет его.

### Q:
Ну я лично ничего не указывал. Виртуалка Гугла, неизвестно, как оно собирали исходники.

### A1:
Да, я уже по именам хостов понял, что это генерация. Ну, такое, засада.

### Q:
```bash
error: externally-managed-environment

× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
    python3-xyz, where xyz is the package you are trying to
    install.

    If you wish to install a non-Debian-packaged Python package,
    create a virtual environment using python3 -m venv path/to/venv.
    Then use path/to/venv/bin/python and path/to/venv/bin/pip. Make
    sure you have python3-full installed.

    If you wish to install a non-Debian packaged Python application,
    it may be easiest to use pipx install xyz, which will manage a
    virtual environment for you. Make sure you have pipx installed.

    See /usr/share/doc/python3.12/README.venv for more information.

note: If you believe this is a mistake, please contact your Python installation or OS distribution provider. You can override this, at the risk of breaking your Python installation or OS, by passing --break-system-packages.
hint: See PEP 668 for the detailed specification.
```
sudo pip3 install psycopg2-binary --break-system-packages

### A1:
system-wide python - это плохо. Там вечно что-то надо под конкретную версию. Я на конде словил. Лучше в папке разработки сразу создать venv и все в него ставить чтобы и систему не сломать, и резвиться как угодно.

