---
layout: post
title: "Railway oriented programming: the survival guide"
date: 2026-01-23 15:50:00 -0300
categories: [ruby, design-patterns]
tags: [ruby-on-rails, design-patterns, railway-oriented-programming, dry-rb]
---

Recentemente, iniciei um novo projeto ambicioso e excitante. Por se tratar de uma aplicação com requisitos altíssimos de uptime e resiliência, sabíamos que a arquitetura tradicional do Rails poderia se tornar um emaranhado de if/else e begin/rescue conforme as regras de negócio crescessem.

Como em todo projeto novo, muitas indefinições ainda pairavam. No entanto, o cronograma não espera: não podíamos simplesmente aguardar que todas as decisões de negócio fossem tomadas para começar a construção. Precisávamos de uma estrutura que fosse, ao mesmo tempo, flexível para mudanças e rígida contra falhas.

Felizmente, meus colegas me apresentaram um padrão de projeto que se encaixou como uma luva: o Railway Oriented Programming (ROP).

Este post é o meu guia pessoal de sobrevivência sobre o ROP. Popularizado por Scott Wlaschin, esse padrão trata o fluxo da aplicação como uma malha ferroviária, onde transformamos funções complexas em uma série de pequenos passos conectados. Se você, como eu, está acostumado com o MVC clássico, prepare-se para mudar a forma como enxerga o fluxo de dados.

