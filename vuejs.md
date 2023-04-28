# Vue.js

Позволяет распределить части сайта (кнопка, список, список задач, список пользователей) в объекты JavaScript со своей вёрсткой и логикой.

```js
<!doctype html>
<html lang="ru">
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <script src="https://unpkg.com/vue@3"></script>
    </head>
    <body>
        <div id="app"></div>
        <script>

            // компоненты - обычные JavaScript объекты
            const WelcomeText = {

                // значения, которые компонент может принимать
                props: ['text']

                // HTML шаблон компонента
                // используется многострочное объявление строки
                template: `
                    <p>{{ text }}</p>`
            }

            // корневой компонент
            const App = {

                // необходимо перечислять используемые внутри компоненты
                components: {
                    WelcomeText
                },

                // объявление "реактивных" переменных
                // изменение переменной сразу отображается
                data () {
                    return {
                        value1: "sup, dude?"
                    }
                }

                // пример передачи значений в компонент + привязка
                // показана передача переменной и строкового литерала
                template: `
                    <h1>Example Application</h1>
                    <WelcomeText :text="value1"/>
                    <WelcomeText :text="'hello, daddy!'"/>`
            }
            // создаём экземпляр приложения
            const app = Vue.createApp(App)

            // соединяем приложение к элементом HTML
            app.mount('#app')
        </script>
    </body>
</html>
```

Можно создавать несколько корневых компонентов и без помех привязывать их в уже существующий сайт.

Также можно разбить приложение на множество файлов под каждый компонент, после чего использовать систему сборки, например Vite, которая преобразует множество файлов в собранные вместе и оптимизированные HTML/CSS/JS.

Создать шаблон приложения с использованием Vite

    npm init vue

После нескольких вопросов запускаем веб-сервер разработки

    npm run dev

Веб-сервер поддерживает горячее обновление изменений. Любое изменение кода вызывает обновление страницы браузера.

После того, как приложение будет готово к публикации, запускаем сборку

    npm run build

Итоговые файлы помещаются в папку `dist`

## TailwindCSS

Фреймворк готовых базовых свойств CSS для простой стилизации элементов через классы.

Для интеграции его в процесс сборки Vue.js установим

    npm install tailwindcss postcss autoprefixer

Создаём шаблонные файлы настроек для TailwindCSS и PostCSS (`-p`)

    npx tailwindcss init -p

Указываем в `tailwind.config.js` расположение всех файлов, где используются классы TailwindCSS, чтобы система сборки создала минимальный CSS только используемых классов

```js
module.exports = {
  content: [
    "./index.html",
    "./src/**/*.{vue,js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

Добавляем в наш исходный код новый файл `./src/index.css`

```js
@tailwind base;
@tailwind components;
@tailwind utilities;
```

Импортируем его в `./src/main.js`

```js
import { createApp } from 'vue'
import App from './App.vue'
import './index.css'

createApp(App).mount('#app')
```

Готово! TailwindCSS будет автоматически отрабатывать при запуске сборки приложения.

## Docker

Следующий пример собирает приложение Vue.js из папки `vue-project` и размещает только папку `dist` на новом слое Docker в веб-сервере Nginx.

```
# multi-stage сборка

# начинаем из образа nodejs
FROM node

USER node
WORKDIR /home/node/app

# копируем информацию о зависимостях отдельно от исходного кода,
# чтобы не устанавливать их каждый раз при изменении проекта
COPY ["vue-project/package.json", "vue-project/package-lock.json*", "."]
RUN npm install

# копируем проект
COPY vue-project .

# запускаем сборку
RUN npm run build

# переключаемся на образ Nginx
FROM nginx

# копируем конфигурацию для работы History mode
COPY default.conf /etc/nginx/conf.d/default.conf

# переносим из первого этапа результат сборки
COPY --from=0 /home/node/vue-project/dist /usr/share/nginx/html
```

default.conf

```
server {
    listen       80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
}
```

Собираем образ

    docker build -t vue-project .

Или если через compose

    docker compose up -d --build

TODO можно доработать сборку со скачиванием исходников через Git

