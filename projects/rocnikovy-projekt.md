---
layout: default
title: Embedded komponenty projektu Deadlock
---

# Ročníkový projekt: Embedded komponenty projektu Deadlock


  - **Študent:** Adam Dej [adam@ksp.sk](mailto:adam@ksp.sk)
  - **Vedúci projektu:** RNDr. Richard Ostertág [ostertag@dcs.fmph.uniba.sk](mailto:ostertag@dcs.fmph.uniba.sk)
  - **Cieľ:** Vytvoriť návrh, vyrobiť embedded hardware a vytvoriť software pre embedded komponenty projektu Deadlock.

Projekt Deadlock (premenovaný z projektu Gate) je systém na ovládanie dverí na základe čítania ISIC-ov. Na tomto projekte spolupracuje viac ľudí. Cieľom môjho ročníkového projektu je vytvoriť embedded hardware (vytvoriť design a postaviť) a software pre komponenty 'Reader' a 'Controller' tohto projektu. Napriek tomu, že na projekte spolupracuje viac ľudí práca na týchto komponentoch je moja a len moja.

## Komponenty

### Reader

Reader je čítačka ISIC kariet ktorú vidí užívateľ. Jej úlohou je prečítať údaje z ISIC-u a oznámiť takú udalosť Controlleru. Reader je tiež zodpovedný za poskytovanie vizuálneho a audio užívateľského rozhrania.

### Controller

Controller je zariadenie ktoré rozhoduje o otvorení jedných dverí na základe informácií z (maximálne dvoch) pripojených Readerov. Ovláda zámok a monitoruje stav otvorenia dverí. Komunikuje so Servrom, z ktorého dostáva momentálnu databázu kariet a zasiela mu logy z rôznych udalostí. Komunikácia je cez jednoduchý UDP protokol. Napájanie je riešené cez Power-over-ethernet z dôvodu minimalizácie náročnosti inštalácie. Konfigurácia a pamäť je riešená Micro-SD kartou. To tiež znamená, že ak dôjde k poruche na Controlleri stačí zobrať záložný kus, vymeniť kartu a dvere fungujú rovnako ako predtým s nulovou námahou.

## Linky

  - [Repozitár s designom HW pre Reader](https://github.com/fmfi-svt-deadlock/reader-hw)
  - [Wiki s dokumentáciu k projektu](https://github.com/fmfi-svt-deadlock/server/wiki)
  - Ostatné repozitáre pribudnú čoskoro
