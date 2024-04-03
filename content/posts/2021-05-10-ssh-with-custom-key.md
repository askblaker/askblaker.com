---
title: "SSH Commands"
date: 2021-05-10T19:34:32+02:00
draft: true
---

# Generate ssh key
```bash
ssh-keygen
```

# SSH with custom key
```bash
ssh root@<ip> -i <ssh-key-path>
```

# SSH with custom key and verbose output
```bash
ssh root@<ip> -i <ssh-key-path> -v
```