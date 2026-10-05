# HackNotes

Sitio de prácticas de seguridad (writeups de laboratorio) — Jekyll + [Minimal Mistakes].

**URL:** https://luisrodriguez19.github.io/hacknotes

## Publicar (GitHub Pages)

1. Crear el repo `hacknotes` en la cuenta `Luisrodriguez19` y hacer push de este contenido.
2. En **Settings → Pages**: *Source* = `Deploy from a branch`, rama `main`, carpeta `/ (root)`.
3. En 1-2 min queda en la URL de arriba.

## Escribir un writeup nuevo

Crear `_posts/AAAA-MM-DD-titulo.md` con front matter:

```yaml
---
title: "Título"
excerpt: "Resumen de una línea."
date: 2026-10-04
categories: [wifi]        # wifi | red | web | ad | host | osint | post
tags: [deauth, wpa2]
---
```

> Regla de oro: **nivel público = sanitizado**. Nada de BSSID/MAC/IPs reales; usar
> placeholders (`AA:BB:CC:DD:EE:FF`). El detalle operativo real vive en el repo privado
> de guías, no aquí.

## Previsualizar en local (opcional)

```powershell
bundle install
bundle exec jekyll serve   # http://localhost:4000/hacknotes
```

## Avatar

Colocar `assets/images/avatar.png` (foto/logo) para la barra lateral.
