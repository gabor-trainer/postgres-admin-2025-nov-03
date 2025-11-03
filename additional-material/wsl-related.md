# WSL 2 Related commands for learning PostgreSQL using wsl

### install (optional)

```bash
wsl --install
```

- if wsl is not starting (with Logon failure: the user has not been granted the requested logon type at this computer.
  Error code: Wsl/Service/CreateInstance/CreateVm/HCS/0x80070569)

  ```bash
  Get-Service vmcompute | Restart-Service
  ```

### list all the installed distributions

```bash
wsl --list --verbose
```

### list available distributions (online)

```bash
wsl --list --online
```

### install for example ubuntu 24.04

```bash
  wsl --install -d Ubuntu-24.04
```

### activate systemd in wsl 2

- edit the file `/etc/wsl.conf` and add the following lines:

```bash
  [boot]
  systemd=true
```

- restart the wsl 2 (not this should be executed in the windows terminal)

```bash
wsl --shutdown
```

- then start the wsl 2 again

```bash
wsl
```
