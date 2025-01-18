<!-- vim-markdown-toc GFM -->

* [Firewall](#firewall)
    * [firewall-cmd](#firewall-cmd)

<!-- vim-markdown-toc -->

# Firewall

## firewall-cmd

```sh
firewall-cmd --list-all

firewall-cmd --add-service=http --permanent
firewall-cmd --add-port=80/tcp --permanent

firewall-cmd --reload
```
