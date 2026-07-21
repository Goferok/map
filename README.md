# Карта районов по коэффициентам

Отдельный статический проект для Vercel. Показывает районы на MapLibre GL и красит их по коэффициенту производства или минимальной сумме сметы.

## Запуск

```bash
npm install
npm run dev
```

## Сборка

```bash
npm run build
```

## Данные

- `public/data/spb_municipalities.geojson` — границы районов.
- `public/data/municipality-settings.json` — коэффициенты, минимальные сметы и признак `worksEnabled`.

Если настройки меняются в интерфейсе, они сохраняются в `localStorage`. Чтобы сделать их общими для всех на хостинге, нажмите `Скачать JSON` и замените файл `public/data/municipality-settings.json`.
