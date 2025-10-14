# Monitor de Links Unifi - n8n Workflow

Workflow automatizado para n8n que monitora periodicamente o status e a saúde dos links WAN dos gateways Unifi através de verificações agendadas, enviando notificações via WhatsApp (Evolution API) sempre que detectar problemas de conectividade, perda de pacotes ou gateways offline.

## Funcionalidades

- Monitoramento automático a cada 30 minutos
- Detecção de problemas de conectividade (state != connected)
- Identificação de perda de pacotes (wanUptime <= 80%)
- Envio de alertas via WhatsApp através da Evolution API
- Sistema inteligente anti-spam: evita envio de alertas duplicados
- Re-envio automático após 2 horas se o problema persistir

## Fluxo de Funcionamento

1. **Busca sites da Unifi**: Consulta a API da Unifi para obter todos os sites configurados
2. **Limpa dados de WANs**: Processa e estrutura as informações dos links WAN
3. **Busca hostname**: Obtém o hostname de cada gateway/host
4. **Filtra problemas**: Identifica links com problemas (desconectados ou com uptime <= 80%)
5. **Verifica alertas anteriores**: Consulta se já foi enviado um alerta igual
6. **Envia no WhatsApp**: Envia alerta apenas se:
   - Há problemas detectados, E
   - A mensagem é diferente da última enviada, OU
   - Passaram mais de 2 horas desde o último alerta

## Pré-requisitos

Antes de começar, certifique-se de ter:
- **n8n** instalado e funcionando
- **Evolution API** instalada e configurada
- Uma **instância da Evolution API** criada e conectada ao WhatsApp

## Requisitos

### 1. Configurar Eventos (Webhooks) na Evolution API

Para garantir o funcionamento correto do workflow, é necessário configurar os eventos na sua instância da Evolution API:

1. Acesse o painel da sua Evolution API
2. Entre na instância que você criou
3. Navegue até **Events** > **Webhook**
4. Configure uma **Webhook URL** (pode ser qualquer URL válida, ex: `https://seu-dominio.com/webhook`)
5. Ative os seguintes eventos:
   - ✅ **CHATS_UPSERT**
   - ✅ **GROUPS_UPSERT**
   - ✅ **MESSAGES_UPSERT**
6. Salve as configurações

**Nota**: A URL é obrigatória para salvar as configurações, mas este workflow não utiliza webhooks. Os eventos garantem que a Evolution API esteja sincronizada e possa buscar mensagens corretamente quando o workflow executar.

### 2. Instalar Node Evolution Community

Instale o pacote da Evolution API diretamente no n8n:

1. Acesse **Settings** no menu do n8n
2. Clique em **Community Nodes**
3. Clique em **Install**
4. No campo de input, digite: `n8n-nodes-evolution-api`
5. Clique em **Install**

Documentação oficial: https://www.npmjs.com/package/n8n-nodes-evolution-api

### 3. Configurar Credenciais da Evolution API

No n8n, configure as credenciais da Evolution API:

1. Acesse **Credentials** no menu do n8n
2. Clique em **Add Credential**
3. Busque por "Evolution API"
4. Preencha com as informações da sua instância Evolution:
   - **Base URL**: URL da sua API Evolution
   - **API Key**: Chave de API da Evolution

**IMPORTANTE**: Após importar o workflow, você precisará configurar as credenciais da Evolution API em **2 nodes**:
- **Buscar_Ultima_Mensagem_Grupo**
- **Enviar_Alerta_WhatsApp**

Para configurar:
1. Clique em cada um desses nodes
2. No campo "Credential to connect with", selecione a credencial da Evolution API que você criou
3. Salve as alterações

### 4. Configurar Variáveis do Workflow

No node **Variaveis** dentro do workflow, você precisa configurar os seguintes parâmetros:

```json
{
  "token_unifi": "<seu-token-api-unifi>",
  "instancia_evo": "<nome-da-instancia-evolution>",
  "contato_alerta": "<numero-que-recebe-alerta@s.whatsapp.net>"
}
```

#### Descrição das Variáveis:

- **token_unifi**: Token de API da Unifi (obtenha em https://unifi.ui.com/api)
- **instancia_evo**: Nome da instância configurada na Evolution API
- **contato_alerta**: JID/LID do contato, grupo ou lista que receberá os alertas (formatos: `5511999999999@s.whatsapp.net`, `120363xxxxx@g.us` ou `{ID}@lid`)

## Como Obter o Token da Unifi

1. Acesse https://unifi.ui.com/api
2. Faça login com sua conta Unifi
3. Clique em **Create New**
4. Escolha um nome para o token
5. Defina o tempo de expiração (expiration)
6. Clique em **Create**
7. Copie o token gerado e cole no campo `token_unifi`

## Como Obter o JID/LID do WhatsApp

Para obter o JID ou LID (identificadores) de um contato ou grupo no WhatsApp:

**Através da Evolution API:**
- **Grupos**: `GET /group/findGroupInfos/{instance}`
- **Contatos**: `GET /chat/whatsappNumbers/{instance}`

**Ou utilize o formato padrão:**
- **JID** - Contato individual: `{DDI}{DDD}{NUMERO}@s.whatsapp.net`
- **JID** - Grupo: `{ID_DO_GRUPO}@g.us`

**Nota**: O workflow suporta tanto JID quanto LID para envio de alertas. Para listas de transmissão, utilize o formato `{ID}@lid` (não há endpoint específico na Evolution API para buscar listas).

## Instalação

1. Importe o arquivo `monitor-link-unifi.json` no seu n8n
2. Configure as credenciais da Evolution API
3. Edite o node **Variaveis** com suas informações
4. Ative o workflow

## Formato dos Alertas

Os alertas são enviados no seguinte formato:

```
‼ Monitoramento de Links GW Unifi ‼

Gateway: [nome-do-gateway]
⚠️ Status: [status]

Gateway: [nome-do-gateway]
wan1 (ISP): Down

Gateway: [nome-do-gateway]
wan2 (ISP): 65 (Perda de pacotes)
```

## Interpretação dos Status

- **Down**: WAN completamente offline (wanUptime entre 0-9%)
- **Perda de pacotes**: WAN com problemas (wanUptime entre 10-80%)
- **Status != connected**: Gateway com problema de conexão

## Estrutura dos Nodes

| Node | Descrição |
|------|-----------|
| Disparador_a_Cada_30min | Trigger que executa o workflow a cada 30 minutos |
| Variaveis | Define as variáveis de configuração |
| Buscar_Sites_Unifi | Consulta a API da Unifi para obter os sites |
| Limpar_Dados_WAN | Processa e limpa os dados das WANs |
| Buscar_Hostname_do_Host | Obtém o hostname de cada gateway |
| Combinar_Hostname_e_WANs | Combina informações de hostname e WANs |
| Filtrar_Problemas_e_Formatar_Alerta | Filtra apenas problemas e formata a mensagem |
| Tem_Problemas? | Verifica se há problemas a reportar |
| Buscar_Ultima_Mensagem_Grupo | Busca a última mensagem enviada |
| Mensagem_Ja_Foi_Enviada? | Verifica se a mensagem já foi enviada |
| Verificar_Tempo_Desde_Ultima_Msg | Calcula o tempo desde a última mensagem |
| Passaram_2_Horas? | Verifica se passaram mais de 2 horas |
| Enviar_Alerta_WhatsApp | Envia o alerta via Evolution API |

## Troubleshooting

### Erro: "Evolution API credentials not found"
- Verifique se as credenciais da Evolution API foram configuradas corretamente no n8n

### Erro: "Invalid X-API-Key"
- Verifique se o token da Unifi está correto e válido

### Mensagens não estão sendo enviadas
- Verifique se a instância da Evolution API está ativa
- Confirme se os JIDs/LIDs dos contatos estão no formato correto
- Verifique os logs do workflow para identificar erros

### Workflow não está executando automaticamente
- Certifique-se de que o workflow está ativo (botão toggle no canto superior direito)
- Verifique se o trigger está configurado corretamente

## Contribuindo

Sinta-se à vontade para abrir issues ou pull requests com melhorias.

## Licença

Este projeto é de código aberto e está disponível sob a licença MIT.

## Versão

**v1.0.0** - Release inicial
