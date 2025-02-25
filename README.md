# Sistema de Ocorrências

## Visão Geral

O sistema de **Ocorrências** tem como objetivo apresentar dados inseridos no banco de dados em tempo real, permitindo a interação do usuário com as ocorrências de problemas. Ele possibilita as seguintes funcionalidades:

- **Reconhecimento das Ocorrências**: Confirmar que uma ocorrência foi visualizada, mas não garante que tenha sido solucionada.
- **Soneca**: Definir um tempo de "soneca" para certas ocorrências, para que o usuário não seja interrompido por problemas irrelevantes.
- **Situação da Ocorrência**: Permitir que o status da ocorrência seja alterado para "solucionada" ou "não solucionada".

## Estrutura do Projeto

### Front-End

O projeto é composto pelos seguintes arquivos principais no front-end:

#### Pasta `frontEnd`

- **`index.html`**: Estrutura da interface da web.

#### Pasta `css`

- **`fonts`**: Contém todas as fontes utilizadas na página.
  - **`fonts.css`**: Arquivo que define as fontes.
  
- **`main.css`**: Arquivo principal de estilo, que integra outros arquivos CSS.
- **`body.css`**: Estilização padrão do corpo da página.
- **`buttons.css`**: Estiliza todos os botões, incluindo animações de hover e focus.
- **`keyFrames.css`**: Contém animações usando keyframes.
- **`listOccurrences.css`**: Estiliza a lista de ocorrências.
- **`manipulationJs.css`**: Estiliza as classes manipuladas por JavaScript.
- **`search.css`**: Estiliza os itens de pesquisa.

#### Pasta `src`

- **`central.js`**: Arquivo principal do JavaScript, integrando os outros scripts.
- **`request.js`**: Função que busca ocorrências e realiza ações como reconhecimento e soneca.
- **`search.js`**: Função que realiza a pesquisa de usinas pelo nome.
- **`setSituation.js`**: Função que altera a situação das ocorrências no banco de dados.
- **`obs.js`**: Arquivo de testes.

---

### Back-End

#### Descrição

Este sistema é construído usando Node.js, com conexão ao banco de dados PostgreSQL e comunicação via WebSocket. Ele é projetado para escutar notificações de inserções na tabela problemas e enviar essas notificações para os clientes conectados via WebSocket. Além disso, fornece endpoints HTTP para consultar e atualizar o status das ocorrências.

### Tecnologias Utilizadas
- **`Node.js`** : Ambiente de execução JavaScript no servidor.
- **`Express`** : Framework para construção da API HTTP.
- **`pg`** : Cliente PostgreSQL para Node.js.
- **`WebSocket`** : Protocolo para comunicação bidirecional entre servidor e cliente.
- **`Cors`** : Permite que a API seja acessada por domínios diferentes.

---
### Requisitos

- **`Node.js`** : Versão 14 ou superior.
- **`Banco de Dados PostgreSQL`**: Configuração local ou remota com a tabela problemas configurada corretamente.
---
## Banco de Dados

A tabela de **problemas** foi criada com a seguinte estrutura:

```sql
CREATE TABLE problemas (
    id SERIAL PRIMARY KEY,        -- Campo de ID com incremento automático
    usuario VARCHAR(255) NOT NULL, -- Nome do usuário/operador
    usina VARCHAR(255) NOT NULL,   -- Nome da usina
    descricao TEXT,                -- Descrição do problema
    situacao BOOLEAN DEFAULT FALSE, -- Indica se a ocorrência foi solucionada ou não
    soneca BOOLEAN DEFAULT FALSE,   -- Indica se a ocorrência foi colocada em "soneca" ou não
    visualizacao BOOLEAN DEFAULT FALSE, -- Indica se a ocorrência foi visualizada
    data TIMESTAMP DEFAULT CURRENT_TIMESTAMP, -- Data e hora que a ocorrência apareceu no banco de dados
    data_visualizacao TIMESTAMP, -- Data e hora da visualização da ocorrência
    data_situacao TIMESTAMP -- Data e hora que a ocorrência foi solucionada 
);
```



## Triggers do Banco de Dados

### 1. Alerta de Nova Inserção
Uma trigger é acionada toda vez que uma nova ocorrência é inserida no banco de dados, enviando uma notificação para um canal e registrando um alerta.

```sql
CREATE OR REPLACE FUNCTION alerta_nova_insercao()
RETURNS TRIGGER AS $$
BEGIN
    RAISE NOTICE 'Nova inserção detectada na tabela problema com ID: %, por % na usina: %', NEW.id, NEW.usuario, NEW.usina;
    PERFORM pg_notify('novo_problema', json_build_object(
        'id', NEW.id, 
        'usuario', COALESCE(NEW.usuario, 'Desconhecido'), 
        'usina', COALESCE(NEW.usina, 'Desconhecida'), 
        'descricao', COALESCE(NEW.descricao, 'Sem descrição')
    )::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Trigger para Inserção de Ocorrência
```sql 
CREATE TRIGGER trigger_alerta_nova_insercao
AFTER INSERT
ON problemas
FOR EACH ROW
EXECUTE FUNCTION alerta_nova_insercao();
```

### 2. Atualização de Data de Visualização
Uma função que atualiza a data de visualização sempre que o campo visualizacao muda para TRUE.

```sql
CREATE OR REPLACE FUNCTION atualizar_data_visualizacao()
RETURNS TRIGGER AS $$
BEGIN
    -- Verifica se o campo visualizacao mudou de false para true
    IF OLD.visualizacao = FALSE AND NEW.visualizacao = TRUE THEN
        NEW.data_visualizacao = NOW(); -- Atualiza o campo com a data e hora atuais
        RAISE NOTICE 'Trigger acionada: data_visualizacao atualizada para %', NEW.data_visualizacao;
    ELSE
        RAISE NOTICE 'Trigger não acionada: visualizacao não mudou para true';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```
Trigger para Atualização da Data de Visualização
```sql
CREATE TRIGGER trigger_atualizar_data_visualizacao
BEFORE UPDATE ON problemas
FOR EACH ROW
EXECUTE FUNCTION atualizar_data_visualizacao();
```

## 3. Atualização de Data de Situação
Uma função que atualiza a data da situação sempre que o campo situacao é alterado.

```sql
CREATE OR REPLACE FUNCTION atualizar_data_situacao()
RETURNS TRIGGER AS $$
BEGIN
    -- Verifica se o campo situacao foi modificado
    IF OLD.situacao IS DISTINCT FROM NEW.situacao THEN
        NEW.data_situacao = NOW(); -- Atualiza o campo com a data e hora atuais
        RAISE NOTICE 'Trigger acionada: data_situacao atualizada para %', NEW.data_situacao;
    ELSE
        RAISE NOTICE 'Trigger não acionada: situacao não foi modificada';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Trigger para Atualização da Data de Situação

```sql
CREATE TRIGGER trigger_atualizar_data_situacao
BEFORE UPDATE ON problemas
FOR EACH ROW
EXECUTE FUNCTION atualizar_data_situacao();
```

### Funcionalidades do Sistema
- **Reconhecimento de Ocorrência:** O sistema permite que o usuário reconheça ocorrências, alterando seu status no banco de dados.
- **Soneca:** O usuário pode definir um status de "soneca" para as ocorrências.
- **Alteração de Situação:** O usuário pode alterar a situação da ocorrência para "solucionada" ou "não solucionada", e o sistema registra a data e hora da mudança.
