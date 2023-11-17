---
layout: post
title: HomePage + Dataview 💛
date: 2023-11-16 00:23:30 +0000
categories: [Obsidian]
tags: [Dataview, Homepage]
---

![img-description](https://source.unsplash.com/8z970S60IN4){: w="600" .shadow }
_Photo by [ainr](https://unsplash.com/@ainr) on [Unsplash](https://unsplash.com)_

## HomePage + Dataview

Некоторое время назад открыл для себя в [Obsidian](https://obsidian.md/) отличное сочетание плагинов [HomePage](https://github.com/mirnovov/obsidian-homepage) и [Dataview](https://github.com/blacksmithgu/obsidian-dataview).

[HomePage](https://github.com/mirnovov/obsidian-homepage) позволяет создать, так называемую, _точку входа_ или _главную страницу_ вашей базы заметок. Если раньше приходилось нужные мне заметки искать вручную по папкам, то сейчас можно разместить ссылки на них на главной странице.

А плагин [Dataview](https://github.com/blacksmithgu/obsidian-dataview) позволяет автоматически собирать и агрегировать заметки по различным параметрам.  
  
Например, я хочу чтобы на главной странице были ссылки на странички, где я веду записи/задачи по активностям. Вот такой запрос я использую, чтобы получить список задач на саморазвитие.

```sql
LIST WITHOUT ID
	file.link + " (Прогресс " + round((length(filter(file.tasks.completed, (t) => t = true))) / (length(file.tasks.text)) * 100, 1) + "%)"
WHERE self-development
SORT file.mtime DESCENDING
LIMIT 5
```

Этот скрипт выгружает страницы с тегом `self-development`, сортирует их по времени изменения и берет последние 5 страниц. Плюсом он считает и отображает процент выполненных задач. Выглядит это примерно так.  
  
![](assets/img/obsidian/dataview_and_homepage.png){: w="500" :.normal :.shadow}  
  
При создании новых страниц с таким же тегом, они автоматически будут появляться на главной странице, а старые перестанут отображаться. Таким образом, сочетание HomePage + Dataview дает пользователю возможность структурировать и организовать заметки таким образом, чтобы они были легко доступны, избавляя от ручной рутинной работы.
  
