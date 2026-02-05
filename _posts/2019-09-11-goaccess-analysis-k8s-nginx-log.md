---

title: GoAccess Analysis of K8s Nginx Log

category: Gist
tags: [Geek]
date: 2019-09-11
---

## Software
- Use this document to update GoAccess to the latest version: https://goaccess.io/download
- Use this document to find the suitable log format: https://github.com/allinurl/goaccess/blob/master/config/goaccess.conf
- Install the `jq` Linux tool
- 
## Analyze K8s Nginx Log
```bash
find */* -type f  -exec jq ".textPayload" \{\} \; | goaccess --log-format='%^ %^ [%h] %^ %^ [%d:%t %^] \"%r\" %^ %b \"%R\" \"%u\" %^ %^ [%v] %^:%^ %^ %T %^ %^' --date-format=%d/%b/%Y --time-format=%H:%M:%S - --with-output-resolver -o out.html
```


