---
title: "Coverage for multiple packages in go"
description: "Coverage for multiple packages in go"
date: "2014-05-06"
categories:
    - "docker"
    - "hugo"
---

## Cover

```
go test -coverprofile=cover.out -coverpkg=profit/... profit/decimal
```
```
echo "mode: set" > coverage.out && cat cover1.out cover2.out | grep -v mode: | sort -r | awk '{if($1 != last) {print $0;last=$1}}' >> coverage.out
```
```
go test -coverprofile=cover2.out -coverpkg=profit/... profit/...
cannot use test profile flag with multiple packages
```
```
go tool cover -html=coverage.out
```
