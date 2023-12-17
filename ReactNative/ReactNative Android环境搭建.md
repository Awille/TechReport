# ReactNative Android环境搭建

官方文档：https://reactnative.cn/docs/0.70/environment-setup

安装node.js

\# 使用nrm工具切换淘宝源
npx nrm use taobao

\# 如果之后需要切换回官方源可使用
npx nrm use npm



npm install -g yarn



创建一个RN项目目录

并在RN项目下创建一个android子目录， 名称必须是android，而且该目录要求是android项目的根目录，即setting.gradle文件就在android目录下， android/setting.gradle  不然RN无法识别



RN项目根目录下创建package.json文件



{
  "name": "MyReactNativeApp",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "start": "yarn react-native start"
  }
}



安装 reactNative



yarn add react-native@0.70.0
yarn add react@16.0.0



Module not found: Can't resolve 'react/jsx-runtime'



https://bobbyhadz.com/blog/react-module-not-found-cant-resolve-react-jsx-runtime

```bash
# 👇️ with NPM
npm install react@latest react-dom@latest

# 👇️ ONLY If you use TypeScript
npm install --save-dev @types/react@latest @types/react-dom@latest

# ----------------------------------------------

# 👇️ with YARN
yarn add react@latest react-dom@latest

# 👇️ ONLY If you use TypeScript
yarn add @types/react@latest @types/react-dom@latest --dev


# for Windows
rd /s /q "node_modules"
del package-lock.json
del -f yarn.lock

# 👇️ clean npm cache
npm cache clean --force

# 👇️ install packages
npm install
```

demo参照：
https://github.com/Awille/ReactJsProject



其他错误参考：
https://blog.csdn.net/u011374875/article/details/119849719