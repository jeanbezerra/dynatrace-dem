# Privacidade, consentimento e Session Replay mobile

Este artigo é um guia técnico, não parecer jurídico. Confirme com Jurídico/Privacidade a base legal, a finalidade, a retenção e a comunicação ao titular. No Brasil, trate LGPD como requisito de produto e segurança.

## Princípios

1. **Minimização:** colete apenas o necessário para uma finalidade registrada.
2. **Consentimento real:** a configuração do SDK deve refletir a escolha da pessoa.
3. **Privacidade na captura:** dado que não sai do dispositivo é mais seguro que dado mascarado depois.
4. **Menor privilégio:** identidade, IP e replay exigem acesso restrito.
5. **Separação de ambientes:** produção e testes não devem misturar usuários nem políticas.
6. **Revisão contínua:** telas e propriedades mudam; testes de masking precisam acompanhar releases.

## Níveis de coleta

| Nível | Efeito prático |
|---|---|
| `OFF` | Nenhuma telemetria RUM é enviada |
| `PERFORMANCE` | Dados automáticos de performance; identificadores randomizados a cada início |
| `USER_BEHAVIOR` | Permite identidade e eventos customizados conforme consentimento |

Crashes/ANRs e replay têm preferências próprias no objeto de privacidade. Não conclua que consentir RUM implica consentir replay.

## Fluxo recomendado de opt-in

```text
App inicia
  → SDK em userOptIn/DTXUserOptIn
  → lê preferência local/CMP
  → se desconhecida: mostra escolhas e mantém coleta desligada
  → aplica DataCollectionLevel + crash + replay
  → registra versão do aviso de privacidade localmente
  → permite alteração posterior
```

Ao mudar consentimento, aplique as novas opções imediatamente. Ao logout, limpe `user.identifier`; quando a separação de contexto exigir, encerre a visita.

## Identidade

- prefira ID interno pseudonimizado/hasheado com estratégia documentada;
- não use e-mail, CPF, telefone ou nome como conveniência;
- não inclua identidade em nomes de view/ação;
- o `identifyUser()` deve ser reaplicado em cada nova sessão autenticada;
- controle acesso a `user.identifier` pelo fieldset sensível.

Política necessária para leitura do fieldset:

```text
ALLOW storage:fieldsets:read WHERE storage:fieldset-name="builtin-sensitive-user-events-and-sessions"
```

## Propriedades

Crie uma allowlist no frontend. Propriedades não permitidas são descartadas, o que é uma defesa adicional. Para cada chave, registre:

- finalidade;
- owner;
- tipo e valores esperados;
- classificação do dado;
- cardinalidade esperada;
- base legal/consentimento;
- data da próxima revisão.

Evite payloads JSON, mensagens cruas, URLs com query string, IDs de pedido e textos digitados. Normalize causas (`timeout`, `declined`, `network`) em vez de enviar a mensagem original.

## Session Replay mobile

Session Replay reconstrói interações e telas e, por isso, amplia o risco. A documentação mobile também oferece replay de contexto anterior a crash. Antes de ativar:

1. aprove a finalidade e o público com acesso;
2. ative RUM e Session Replay no frontend;
3. escolha uma taxa pequena e representativa;
4. configure opt-in de replay/crash replay no app;
5. aplique masking mais seguro por padrão;
6. teste todos os componentes e estados sensíveis;
7. só depois aumente a captura.

### Android

O masking padrão mais restritivo é **Safest**, cobrindo campos editáveis, imagens, labels, WebViews e switches. Há níveis Safe/Custom. Em Compose, marque componentes específicos:

```kotlin
import com.dynatrace.agent.compose.api.dtMask

Text(
    text = customerSensitiveText,
    modifier = Modifier.dtMask()
)
```

Views também podem usar a tag `data-dtrum-mask` ou APIs de configuração específicas.

### iOS

Use o módulo de Session Replay e as APIs de masking do SDK. Valide UIKit e SwiftUI separadamente. Session Replay não é suportado em tvOS.

### Casos que exigem teste explícito

- login, recuperação de senha e MFA;
- documentos, CPF e telefone;
- saldo, fatura e extrato;
- cartão, CVV e pagamento;
- chat, anexos e câmera;
- WebView e conteúdo remoto;
- notificações e overlays;
- telas offline/erro;
- conteúdo carregado depois de uma feature flag.

## Controle de acesso e auditoria

- Separe permissão de replay com masking da permissão sem masking.
- Restrinja exportação, compartilhamento e acesso a campos sensíveis.
- Use grupos por função e prazo.
- Audite acessos e mudanças de configuração.
- Não copie replay/PII para tickets ou canais de chat; use links com autorização.

## Retenção

A documentação consultada indica retenção padrão de 35 dias para `user.events`, `user.sessions` e user replays no Latest Dynatrace, sujeita ao contrato e às opções de retenção estendida. Retenção técnica não substitui uma política de descarte e finalidade.

## Checklist antes da produção

- [ ] aviso e consentimento aprovados;
- [ ] RUM, crash e replay têm escolhas independentes quando necessário;
- [ ] opt-out testado desde primeira instalação e após upgrade;
- [ ] nenhum identificador direto em `identifyUser`;
- [ ] allowlist de propriedades revisada;
- [ ] masking validado em device real e ambos os temas/orientações;
- [ ] grupos e fieldsets seguem menor privilégio;
- [ ] retenção e procedimento de atendimento ao titular documentados;
- [ ] custo/sampling aprovado;
- [ ] teste de regressão de privacidade no release gate.

## Referências oficiais

- [Privacidade em frontends mobile](https://docs.dynatrace.com/docs/observe/digital-experience/rum/mobile-frontends/data-privacy)
- [Session Replay mobile](https://docs.dynatrace.com/docs/observe/digital-experience/session-replay-latest/configure-session-replay-mobile)
- [Session Replay Android](https://docs.dynatrace.com/docs/observe/digital-experience/session-replay-latest/configure-session-replay-mobile/latest-session-replay-android)
- [Dados pessoais capturados](https://docs.dynatrace.com/docs/manage/data-privacy-and-security/data-privacy/personal-data-captured-by-dynatrace)
- [Níveis de proteção](https://docs.dynatrace.com/docs/manage/data-privacy-and-security/data-privacy/levels-of-data-protection)
- [Retenção](https://docs.dynatrace.com/docs/manage/data-privacy-and-security/data-privacy/data-retention-periods)

