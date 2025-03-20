---
date: "{{ .Date | site.time | date: '%Y-%m-%d %H:%M:%S %z' }}"
draft: true
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
---
