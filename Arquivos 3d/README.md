# Modelagem e Impressão 3D

Este diretório contém todos os arquivos necessários para a fabricação das peças estruturais, suportes de sensores e cases do projeto.

## Softwares Utilizados

As peças foram desenvolvidas e preparadas utilizando as seguintes ferramentas:

* **[Autodesk Fusion 360](https://www.autodesk.com/products/fusion-360/overview):** Utilizado para a modelagem paramétrica e design das peças (`.f3d`).
* **[Orca Slicer](https://github.com/SoftFever/OrcaSlicer):** Utilizado para o fatiamento e geração do G-Code para a impressora.

## Estrutura dos Arquivos

* **`/STL`**: Arquivos de malha prontos para serem importados no fatiador.
* **`/Fusion360`** (ou `.f3d`): Arquivos editáveis originais do projeto.
* **`/Gcode`** (Opcional): Arquivos já fatiados para a impressora específica do projeto.

## Lista de Peças

Abaixo estão os arquivos de malha prontos para impressão disponíveis nesta pasta:

### Estrutura Principal
* **`Fuselagem.stl`**: Corpo principal da aeronave, projetado para servir de base e acomodar a eletrônica central.
* **`Bico.stl`**: Parte frontal (nariz) do avião, onde também está localizada a porta. Esconde partes da eletrônica, como buzzer e pilhas.
* **`Conexão.stl`**: Elemento estrutural para junção das partes do corpo, incluindo locais para fixação da válvula para pressurizar e botão para ligar e desligar.

### Interior e Suportes
* **`Piso parte 1.stl`** e **`Piso parte2.stl`**: Base interna plana para fixação segura do ESP32, placa de desenvolvimento e sensores.
* **`tampao.stl`**: Peça de fechamento/acabamento para a parte traseira do avião.

### Mecanismos Móveis
* **`Porta.stl`**: Componente móvel acionado pelo servo motor (simulação de porta de emergência).
