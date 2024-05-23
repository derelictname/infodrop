# tailwindcss

Позволяет задавать стили не через CSS файлы, а через классы в тегах. При этом классы задают сразу несколько параметров и упрощают создание внешнего вида. Также многие JavaScript фреймворки (VueJS, Alpine.js) позволяют программно управлять стилями, меняя внешний вид тегов в зависимости от действий пользователя или состояния интерфейса (переключение вкладок, состояние кнопок, появление тегов).

Установка

    npm install -D tailwindcss@latest daisyui@latest
    npx tailwindcss init

- daisyui - плагин для tailwindcss, предоставляющий готовые комбинации стилей

Указываем в настройках расположение файлов с классами TailwindCSS для сканирования

```js
module.exports = {
  content: ["*.{html,js}"],
  theme: {
    extend: {},
  },
  plugins: [require("daisyui")],
};
```

Создаём CSS файл с командами TailwindCSS, которые он заменит на стили

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

Подключаем CSS файл и используем стили

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link href="output.css" rel="stylesheet" />
  </head>
  <body>
    <h1 class="text-3xl font-bold underline">Hello world!</h1>
    <div class="button">123</div>
  </body>
</html>
```

Запускаем создание файла CSS с готовыми стилями

    npx tailwindcss -i input.css -o tailwind.css --watch

Теперь TailwindCSS будет сканировать классы в коде и создавать CSS файл с только необходимыми стилями.

Примеры использования классов <https://pagedone.io/>

## PostCSS

Плагин PostCSS используется различными сборщиками для обработки CSS файлов с помощью алгоритма AST. TailwindCSS в таком случае выступает плагином для PostCSS.

    npm create vite@latest
    npm install -D tailwindcss@latest postcss@latest autoprefixer@latest daisyui@latest
    npx tailwindcss init -p

- autoprefixer - плагин postcss для добавления приставок в имена CSS настроек

Vite обнаружит файл настроек PostCSS и задействует плагин.

При использовании сборщика файлы CSS можно подключать прямо в JavaScript коде. Также в случае с TailwindCSS исключается этап создания CSS файла. Необходимо подключить стили в `./src/main.js` таким образом

```js
import { createApp } from "vue";
import App from "./App.vue";
import "tailwindcss/tailwind.css";

createApp(App).mount("#app");
```

TailwindCSS будет автоматически выполняться при сборке

    npm run dev

## prose

Если имеется HTML код, к которому нельзя применить классы (например, полученный через преобразование из Markdown), то можно применить к родительскому тегу класс `prose` и он стилизует все дочерние элементы.

Для использования `prose` необходимо установить плагин

    npm install -D @tailwindcss/typography
