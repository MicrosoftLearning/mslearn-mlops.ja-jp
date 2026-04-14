---
title: オンラインでホストされる手順
permalink: index.html
layout: home
---

# MLOps の課題

このリポジトリには、Azure Machine Learning を使用したエンドツーエンドの機械学習の運用 (MLOps) に関する実践的な課題が含まれています。

演習を完了するには、Microsoft Azure のサブスクリプションが必要です。 講師からサブスクリプションが提供されていない場合は、[https://azure.microsoft.com](https://azure.microsoft.com/) で無料の試用版にサインアップできます。

## ラボ

{% assign labs = site.pages | where_exp:"page", "page.url contains '/docs'" %}
{% for activity in labs  %}
<hr>
### [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }})

{{ activity.lab.description }}

{% endfor %}