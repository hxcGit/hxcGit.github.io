# Npm 使用


<!--more-->

### 一、 npm install 参数

#### 1.1 npm install 安装单个 packageName 包

- **无参数 默认情况**

```sh
npm install packageName
```

默认情况下，`不加参数`。会安装包，并将依赖包的名称添加 package.json 中的 dependencies 字段。

- **--save 参数**

```sh
npm install --save packageName
```

添加`--save`参数，与默认情况效果相同。会安装包，并将依赖包的名称添加到 package.json 中的 dependencies 字段。

- **--save-dev 参数**

```sh
npm install --save-dev packageName
```

添加`--save-dev`参数，会安装包，并将依赖包的名称添加到 package.json 中的 devDependencies 字段。

> 安装某个包时，这个包中 package.json 的 dependencies 字段中的依赖会被自动安装，而 devDependencies 字段中的依赖不会被安装。

#### 1.2 npm install 初始化项目

---

- **无参数 直接初始化**

```sh
npm install
```

我们常用 npm install 初始化项目，安装项目所需的依赖。但更深入的细节是：直接使用 npm install 时，项目 package.json 中 dependencies 字段和 devDependencies 字段中的依赖包都会被安装。

- **--production 参数**

```sh
npm install --production
```

添加`--production`安装项目所需的依赖时，只有 dependencies 字段中的依赖包会被安装，devDependencies 中的依赖包不会被安装。

- **--only=dev 参数**

```sh
npm install --only=dev
```

添加`--only=dev`安装项目所需依赖时，只有 devDependencies 字段中的依赖包会被安装，dependencies 字段中的依赖包不会被安装。与添加--production 的效果刚好相反。

> 还有一个参数：--dev，它的效果与--only=dev 相同，但已经被废弃，请使用--only=dev 代替。

