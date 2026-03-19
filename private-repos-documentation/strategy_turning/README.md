# lucas152112/strategy_turning

**Description**: No description

**Primary Language**: Go

**Created**: 2024-10-05T19:21:10Z
**Last Updated**: 2025-12-11T10:02:27Z
**Stars**: 0 | **Forks**: 0
**Archived**: False

## Language Statistics

| Language | Bytes | Percentage |
|----------|-------|------------|
| Go | 338,536 | 60.5% |
| Shell | 145,164 | 25.9% |
| Python | 48,627 | 8.7% |
| HTML | 19,209 | 3.4% |
| JavaScript | 7,833 | 1.4% |
| Makefile | 366 | 0.1% |
| Dockerfile | 225 | 0.0% |

## README 完整性檢查

⚠️ **README 不完整** - 缺少以下部分：
  - 系統架構
  - 技術框架
  - 程式語言

**建議補充內容：**

### 🏗️ 系統架構
*在此描述系統架構...*

### 🛠️ 技術框架
*在此列出使用的技術框架...*

### 💻 程式語言
- 主要語言: Go


*檢查時間: 2026-03-19 14:26:57*

## Repository Information

- **GitHub URL**: https://github.com/lucas152112/strategy_turning
- **SSH URL**: git@github.com:lucas152112/strategy_turning.git
- **Clone URL**: https://github.com/lucas152112/strategy_turning.git

## System Architecture

### Overview

This project uses a modular architecture with the following technology stack:

| Language | Bytes | Percentage |
|----------|-------|------------|
| Go | 338,536 | 60.5% |
| Shell | 145,164 | 25.9% |
| Python | 48,627 | 8.7% |
| HTML | 19,209 | 3.4% |
| JavaScript | 7,833 | 1.4% |
| Makefile | 366 | 0.1% |
| Dockerfile | 225 | 0.0% |

### Key Components

1. **Core Application** - Main business logic and functionality
2. **Data Layer** - Database and data access components
3. **API Layer** - RESTful or GraphQL interfaces
4. **Client Interface** - User-facing applications or services

### Deployment

- **Containerization**: Docker-based deployment
- **Orchestration**: Kubernetes for scalable deployment
- **CI/CD**: Automated testing and deployment pipeline

### Dependencies

- See package manager files for detailed dependencies
## README Content

```
# Go Gin Example [![rcard](https://goreportcard.com/badge/github.com/EDDYCJY/go-gin-example)](https://goreportcard.com/report/github.com/EDDYCJY/go-gin-example) [![GoDoc](http://img.shields.io/badge/go-documentation-blue.svg?style=flat-square)](https://godoc.org/github.com/EDDYCJY/go-gin-example) [![License](http://img.shields.io/badge/license-mit-blue.svg?style=flat-square)](https://raw.githubusercontent.com/EDDYCJY/go-gin-example/master/LICENSE)

An example of gin contains many useful features

[简体中文](https://github.com/EDDYCJY/go-gin-example/blob/master/README_ZH.md)

## Installation
```
$ go get github.com/EDDYCJY/go-gin-example
```

## How to run

### Required

- Mysql
- Redis

### Ready

Create a **blog database** and import [SQL](https://github.com/EDDYCJY/go-gin-example/blob/master/docs/sql/blog.sql)

### Conf

You should modify `conf/app.ini`

```
[database]
Type = mysql
User = root
Password =
Host = 127.0.0.1:3306
Name = blog
TablePrefix = blog_

[redis]
Host = 127.0.0.1:6379
Password =
MaxIdle = 30
MaxActive = 30
IdleTimeout = 200
...
```

### Run
```
$ cd $GOPATH/src/go-gin-example

$ go run main.go 
```

Project information and existing API

```
[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:	export GIN_MODE=release
 - using code:	gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /auth                     --> github.com/EDDYCJY/go-gin-example/routers/api.GetAuth (3 handlers)
[GIN-debug] GET    /swagger/*any             --> github.com/EDDYCJY/go-gin-example/vendor/github.com/swaggo/gin-swagger.WrapHandler.func1 (3 handlers)
[GIN-debug] GET    /api/v1/tags              --> github.com/EDDYCJY/go-gin-example/routers/api/v1.GetTags (4 handlers)
[GIN-debug] POST   /api/v1/tags              --> github.com/EDDYCJY/go-gin-example/routers/api/v1.AddTag (4 handlers)
[GIN-debug] PUT    /api/v1/tags/:id          --> github.com/EDDYCJY/go-gin-example/routers/api/v1.EditTag (4 handlers)
[GIN-debug] DELETE /api/v1/tag
```
