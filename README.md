# app

## Project setup
```
npm install
```

### Compiles and hot-reloads for development
```
npm run serve
```

### Compiles and minifies for production
```
npm run build
```

### Lints and fixes files
```
npm run lint
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).


### common problem
@monaco-editor/loader的版本应该直接控制在 1.2.0，使用^副号，可能出现版本浮动到1.2.8，则会出现找不到getModeId()这个函数的情况。