
# Criação de usuários no Heimdall v2!
Dentro do ambiente da Zoop, um marketplace pode ter a necessidade de permitir que terceiros (normalmente funcionários ou sócios) tenham acesso ao painel do dashboard para fins de visualização ou gerenciamento do marketplace ou de um seller. Visando isso, foi desenvolvida a feature de criação de usuários, onde um marketplace pode realizar o pré cadastro de usuário, enviando um convite para que o mesmo possa contribuir com as atividades do marketplace ou de um seller específico.

O controle de usuários do Heimdall utiliza o *AWS DynamoDB* como base de dados tendo as tabelas *heimdall-users-{nome_do_ambiente}* para os usuários e *heimdall-v2-permissions-{nome_do_ambiente}* para as permissões do mesmo.  
  

### Os usuarios podem ser criados a partir de duas rotas:

> /v2/marketplaces/{marketplace_id}/users
- Cria usuarios com acesso ao marketplace selecionado
> 
> /v2/marketplaces/{marketplace_id}/sellers/{seller_id}users

- Cria usuarios com acesso ao seller selecionado.
  
  

Exemplo de body - json.
```
{
      "username": "exemplo@zoop.com.br"  
      "group":  "EXAMPLE_GROUP_V2"  
      "first_name":  "Usuario"  
      "last_name":  "Valido Heimdall V2"
}
```


  
```mermaid
  graph TD  
  
    %% PRE-LAUNCH SETUP" FLOW CHART SHAPES  
    START["cria<br>usuario"]  
    EXIST_MKTPLC?{Marketplace<br>existe?}  
    EXIST_USER?{usuario ja<br>existe na<br>base?}  
    GENERATE_ID[gera Id<br>de usuario]  
    USER_FOR_SELLER?{usuario<br>esta sendo<br>criado para<br>seller?}  
    SELLER_IS_VALID?{seller eh<br>valido?}  
    PERSIST_USER[/persiste<br>permissoes<br>de usuario/]  
    UPDATE_USER_DFLAG_0[/persiste usuario<br>com status 'aguardando<br>confirmacao de registro'<br>-dflag=0-/]  
    CREATE_SESSION(cria<br>sessao)  
    UPDATE_USER_MAIL[/atualiza usuario com<br>dados do convite<br> - id de sessao e<br>status requested -/]  
    SEND_MAIL(envia convite para <br>usuario cadastrado.)  
    END[FIM]  
  
    %% CREATE FLOW CHART  
    subgraph FLUXO DE CRIAÇÃO DE USUARIO  
    START --> EXIST_MKTPLC?  
    EXIST_MKTPLC? --> |sim| EXIST_USER?  
    EXIST_USER? -->|nao| GENERATE_ID  
    EXIST_MKTPLC? --> |sim| END  
    EXIST_USER? --> |nao| END  
    USER_FOR_SELLER? --> |nao| PERSIST_USER  
    GENERATE_ID --> USER_FOR_SELLER?  
    USER_FOR_SELLER? --> |sim| SELLER_IS_VALID?  
    SELLER_IS_VALID? --> |sim| PERSIST_USER  
    PERSIST_USER --> UPDATE_USER_DFLAG_0  
    SELLER_IS_VALID? --> |nao| END  
    UPDATE_USER_DFLAG_0 --> CREATE_SESSION  
    CREATE_SESSION --> UPDATE_USER_MAIL  
    UPDATE_USER_MAIL --> SEND_MAIL  
    SEND_MAIL --> END  
    end  
```


## Fluxo de criacao de usuario.

Ao criar um novo usuário, Inicialmente é verificada a existência do marketplace na base. Após confirmação, verifica se já não existe associado a esse marketplace o nome de usuário que está se tentando criar. Se o usuário estiver sendo criado para um seller, é verificado se o seller é válido (Obs.: se alguma das verificações não der o resultado esperado, o processo é cancelado e o usuário não é criado).

O usuário é persistido com status “aguardando confirmação de registro” (dflag=0). As permissões do usuário também são persistidas na base.

É adicionado ao usuário na base um histórico de convites contendo um campo com informações do envio de “convite de confirmação de usuário” contendo o Id da sessão criada (para que posteriormente o usuário possa ser confirmado) e com type “requested”. O convite de acesso a plataforma é [enviado para o email cadastrado ](#ancora1) <a id="ancora2"></a> contendo um token de acesso a sessão, tendo como resposta 201: created, com um json como o do exemplo.
```
{
      "created_at":  200900000001,  
      "dflag":  0  
      "marketplace_id:  "0162893710a6495e86542eeff192baa1"  
      "origin":  "heimdall"  
      "profile":  {
          "first_name":  "Usuario"
          "last_name":  "Valido Heimdall V2"
     }
}
```
Após essa etapa, o futuro usuário receberá uma email o convidando a criar uma conta no Checkout Marketplace.

Ao confirmar a criação de conta, sua solicitação irá para a rota `/v2/marketplaces/{marketplace_id}/users/{user_id}/confirm-invitation`

Obs.: Necessita de um header de Authorization do tipo Bearer com o Token da sessão de criação do usuário(enviada dentro do email de ativação).

exemplo de body - json.
  
  
```
{
      "password":  "Zoop@",
      "confirm_password":  "Zoop@"
}
```

``` mermaid
  graph TD  
  
    %% PRE-LAUNCH SETUP" FLOW CHART SHAPES  
    RECEIVED_EMAIL["validar<br>usuario"]  
    VALID_SESSION?{sessao<br>eh valida?}  
    VALID_MARKETPLACE?{marketplace<br>eh valido?}  
    END_SESSION[/encerra sessao/]  
    UPDATE_USER_DFLAG_1[/atualiza usuario<br>para status ativo<br> -dflag=1-/]  
    INVITE_CONSUMED[/atualiza usuario com<br> novos dados do convite<br>-status=consumed-/]  
    PASSWORD_PERSIST[/persiste <br>nova senha/]  
    FINISH(fim)  
    
    %% CREATE FLOW CHART  
    subgraph FLUXO DE VALIDAÇÃO DE USUARIO  
    RECEIVED_EMAIL --> VALID_SESSION?  
    VALID_SESSION? --> |sim| VALID_MARKETPLACE?  
    VALID_SESSION? --> |nao| FINISH  
    VALID_MARKETPLACE? --> |sim| END_SESSION  
    VALID_MARKETPLACE? --> |nao| FINISH  
    END_SESSION --> UPDATE_USER_DFLAG_1  
    UPDATE_USER_DFLAG_1 --> INVITE_CONSUMED  
    INVITE_CONSUMED --> PASSWORD_PERSIST  
    PASSWORD_PERSIST --> FINISH  
    end  
```


Nesse ponto, será verificada a validade da sessão e do marketplace. A sessão é encerrada (excluída da base) e o usuário tem seu status atualizado para “registro ativo” (dflag=1), é adicionado ao histórico de convites uma ocorrência com o id da sessão com o status “consumed” e tem sua nova senha adicionada a seu registro, retornando 204.
  
  
<a id="ancora1"></a>  
## O Envio do e-mail de convite 

Para o envio, são adicionados ao e-mail o id marketplace, id do usuário criado, o nome de usuário (que também é o endereço de email), um template de e mail e o token de acesso a sessão. É verificada a existência do marketplace na base e caso exista, são trazidos dados do marketplace para o preenchimento do e-mail e enviados a um lambda que dispara o e-mail. [voltar](#ancora2)

