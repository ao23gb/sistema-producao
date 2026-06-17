# Contexto do projeto: Sistema de Controle de Produção

> Use este documento como ponto de partida. Ele resume tudo o que já foi definido sobre o sistema: módulos, telas, regras de negócio, modelo de dados e arquitetura técnica recomendada. Construa o sistema seguindo esta especificação, módulo por módulo, perguntando antes de tomar decisões que não estejam cobertas aqui.

## Visão geral

Sistema web de controle de produção, com suporte a múltiplos usuários simultâneos (estimativa de 10 a 50) e perfis de acesso diferenciados (ex.: administrador, colaborador, almoxarifado). Organizado em cinco módulos: Usuários e Permissões, Cadastro, Estoque, Produção (Kanban) e Em Uso.

## Arquitetura técnica recomendada

- Backend: Laravel (PHP)
- Painel administrativo (Cadastro/Estoque): Filament
- Quadro Kanban: Livewire ou Vue.js
- Banco de dados: MySQL ou PostgreSQL
- Autenticação: sistema nativo do Laravel
- Hospedagem (fase final): VPS com Laravel Forge

## Módulos e telas

### 1. Usuários e Permissões
Controle de login e perfis de acesso, definindo o que cada perfil pode ver/fazer (cadastrar, movimentar estoque, gerenciar Kanban, visualizar relatórios).

### 2. Cadastro
- **Colaboradores:** nome, login, senha (com hash), perfil de acesso, opção de restringir acesso à movimentação de almoxarifado.
- **Insumos:** nome, código ID, código interno, código de barras, indicador de produto único, unidade de medida.
- **Materiais:** nome, foto, código ID/interno/barras, tipo (MDF ou MDP), espessura em mm, cor, valor de custo.
- **Produtos:** nome, quantidade de peças por caixa, lista de materiais utilizados (com quantidade), lista de insumos utilizados (com quantidade), sequência de etapas de produção (com opção de reordenar/remover), custo unitário calculado automaticamente.
- **Etapas do Kanban:** nome (ex.: Fila, Cortando, Fitagem, Acabamento, Prontos, Concluído, Tapeçaria), ordem padrão, indicador de ativa/inativa. Possível criar novas etapas.

Regra importante: o custo unitário do produto deve recalcular automaticamente sempre que materiais, insumos ou suas quantidades mudarem. Cada produto pode ter sua própria sequência de etapas, mesmo usando etapas do Kanban geral.

### 3. Estoque
- **Controle de estoque:** tela somente leitura, mostrando saldo total e quantidade aguardando entrega de cada insumo/material.
- **Entrada de insumos:** fluxo em duas etapas — pré-entrada (pedido, status "Aguardando entrega", sem somar ao estoque) e confirmação da entrada (informar quantidade real recebida, que aí sim soma ao estoque total). Histórico filtrável por período.
- **Movimentação de almoxarifado:** acesso restrito a colaboradores autorizados. Registra saída, ajuste ou devolução de insumos/materiais, com colaborador responsável, quantidade e campo de observação (obrigatório em ocorrências fora do padrão).

### 4. Produção (Kanban)
Quadro com colunas por etapa, cartões representando ordens de produção (produto, quantidade, data). Ao mover um cartão entre colunas, registrar no histórico a data/hora de entrada e saída de cada etapa. Permitir criar nova ordem de produção escolhendo produto e quantidade. Filtro por produto.

### 5. Em Uso
- **Controle de uso:** materiais/insumos atribuídos a cada colaborador, com opção de "Baixa" (informando que o item se esgotou) e campo de observação.
- **Visualização de uso:** mesma informação, somente leitura, para usuários com permissão de visualizar relatórios.

## Modelo de dados (entidades principais)

**perfis_acesso** — id (PK), nome_perfil, pode_cadastrar, pode_movimentar_estoque, pode_gerenciar_kanban, pode_visualizar_relatorios

**usuarios** — id (PK), nome, login, senha_hash, perfil_id (FK), ativo, criado_em

**colaboradores** — id (PK), nome, login, senha_hash, perfil_id (FK), restringir_estoque

**insumos** — id (PK), nome, codigo_id, codigo_interno, codigo_barras, produto_unico, unidade_medida

**materiais** — id (PK), nome, foto_url, codigo_id, codigo_interno, codigo_barras, tipo_material, espessura_mm, cor, valor_custo

**produtos** — id (PK), nome, custo_unitario, custo_caixa, qtd_pecas_por_caixa

**produto_materiais** — id (PK), produto_id (FK), material_id (FK), quantidade

**produto_insumos** — id (PK), produto_id (FK), insumo_id (FK), quantidade

**etapas_kanban** — id (PK), nome_etapa, ordem, ativa

**produto_etapas** — id (PK), produto_id (FK), etapa_id (FK), ordem_no_produto

**estoque** — id (PK), insumo_id (FK), material_id (FK), estoque_total, aguardando_entrega, atualizado_em

**entradas** — id (PK), insumo_id (FK), fornecedor, status, quantidade_pedida, quantidade_recebida, usuario_id (FK), data_pedido, data_confirmacao

**movimentacoes_almoxarifado** — id (PK), colaborador_id (FK), insumo_id (FK), material_id (FK), quantidade, tipo_movimentacao, observacao, usuario_id (FK), criado_em

**ordens_producao** — id (PK), produto_id (FK), quantidade, etapa_atual_id (FK), status_geral, criado_por (FK), criado_em, concluido_em

**historico_etapas_producao** — id (PK), ordem_producao_id (FK), etapa_id (FK), entrou_em, saiu_em, movido_por (FK)

**materiais_em_uso** — id (PK), colaborador_id (FK), insumo_id (FK), material_id (FK), quantidade_atribuida, status, observacao_baixa, atribuido_em, baixado_em

## Pontos em aberto (confirmar durante a construção)

- O que acontece com ordens de produção em andamento quando a sequência de etapas de um produto é alterada (a recomendação inicial é não afetar retroativamente o histórico).
- Regra exata de como a baixa de um material "em uso" reflete (ou não) no estoque, dependendo do tipo de item.

## Ordem sugerida de construção (MVP)

1. Autenticação e perfis de acesso
2. Cadastro (colaboradores, insumos, materiais, produtos, etapas)
3. Estoque (controle, entrada, movimentação)
4. Produção (Kanban)
5. Em Uso (controle e visualização)
