# SSH

## 同时在多台主机上执行相同的命令

```bash
echo "uptime" | tee >(ssh -T root@servera) >(ssh -T root@serverb) >/dev/null
```
