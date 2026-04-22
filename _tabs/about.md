---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

<div id="logo-banner" markdown="1">
[![Trysil](/assets/img/trysil-logo-light.png){: .normal }](https://github.com/davidlastrucci/Trysil)
[![Trysil](/assets/img/trysil-logo-dark.png){: .normal }](https://github.com/davidlastrucci/Trysil)
</div>

<style>
#logo-banner { margin: 1.5rem 0; }
#logo-banner > p { margin: 0; }
#logo-banner a.img-link { display: block; }
#logo-banner a.img-link img { display: block; margin: 0 auto; max-width: 75%; height: auto; }
html[data-mode="light"] #logo-banner a.img-link:nth-of-type(2),
html:not([data-mode]) #logo-banner a.img-link:nth-of-type(2) { display: none; }
html[data-mode="dark"] #logo-banner a.img-link:nth-of-type(1) { display: none; }
</style>

## About this blog

This is the official blog of [Trysil](https://github.com/davidlastrucci/Trysil),
an ORM framework for Delphi.

Here you'll find tutorials, deep dives into the internals, release notes,
and the reasoning behind design decisions — both what worked and what didn't.

## About Trysil

Trysil is an attribute-driven ORM for Delphi supporting SQLite, PostgreSQL, FirebirdSQL,
and SQL Server through FireDAC. It ships with a JSON module for serialization and an
HTTP module for REST APIs with multi-tenant support.

Supported Delphi versions: 10.3 Rio through 13 Florence.

## About the author

Trysil is developed and maintained by **David Lastrucci**.

- GitHub: [@davidlastrucci](https://github.com/davidlastrucci)
- Main repository: [Trysil](https://github.com/davidlastrucci/Trysil)
