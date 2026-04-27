# 🛡️ Case Study: Client-Side Authorization Bypass via Response Manipulation

> **Disclosure responsável** — O nome da plataforma afetada foi omitido intencionalmente. Os detalhes técnicos são compartilhados exclusivamente para fins educacionais e de conscientização em segurança.

---

## 📋 Índice

- [Visão Geral](#-visão-geral)
- [Ambiente e Ferramentas](#️-ambiente-e-ferramentas)
- [Análise da Vulnerabilidade](#-análise-da-vulnerabilidade)
- [Reprodução (PoC)](#-reprodução-poc)
- [Impacto](#-impacto)
- [Recomendações](#-recomendações)
- [Lições Aprendidas](#-lições-aprendidas)

---

## 🔎 Visão Geral

| Campo            | Detalhe                                          |
|------------------|--------------------------------------------------|
| **Categoria**    | Business Logic / Broken Access Control           |
| **CWE**          | CWE-602 – Client-Side Enforcement of Server-Side Security |
| **Superfície**   | Resposta HTTP do endpoint de autenticação        |
| **Impacto**      | Bypass de restrições de plano (Free → Premium)   |
| **Severidade**   | Média *(limitada à camada de apresentação)*      |
| **Status**       | Documentado para fins educacionais               |

Uma plataforma de música controlava o acesso a funcionalidades Premium **exclusivamente** com base em um campo JSON retornado pelo servidor no momento do login. Embora tokens JWT fossem utilizados para autenticação de sessão, o front-end tomava decisões de autorização com base em um payload JSON mutável — sem qualquer assinatura ou validação adicional.

---

## 🛠️ Ambiente e Ferramentas

```
Burp Suite Community Edition  →  Interceptação e manipulação de tráfego HTTP/2
Chromium                      →  Análise de comportamento da interface (DevTools)
```

---

## 🔍 Análise da Vulnerabilidade

### Fluxo legítimo

```
[Cliente]  →  POST /auth/signin         →  [Servidor]
[Servidor] →  200 OK + JWT + JSON body  →  [Cliente]
[Cliente]  →  Lê campo "plan" do JSON  →  Habilita/Desabilita UI
```

### Resposta original do servidor

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "usr_000",
    "email": "user@example.com",
    "plan": "free"
  }
}
```

### Evidências

<table>
  <tr>
    <td align="center" width="50%">
      <img src="https://github.com/user-attachments/assets/d26c3809-2fcf-4242-ac06-3a9de7b6997b" alt="Resposta interceptada com plan alterado para plus" width="100%"/>
      <br/>
      <sub><b>Fig. 1</b> — Resposta do servidor após manipulação: <code>"plan": "plus"</code></sub>
    </td>
    <td align="center" width="50%">
      <img src="https://github.com/user-attachments/assets/37674400-c859-425e-8a4c-d7e3fef56a7b" alt="Burp Suite mostrando campo plan free selecionado" width="100%"/>
      <br/>
      <sub><b>Fig. 2</b> — Burp Suite: campo <code>"plan": "free"</code> identificado na resposta original</sub>
    </td>
  </tr>
</table>

---

## 🚀 Reprodução (PoC)

> ⚠️ **Aviso:** Este conteúdo é estritamente educacional. Reproduzir esta técnica em sistemas sem autorização explícita é ilegal e antiético.

### Passo 1 — Interceptar a resposta de login

No Burp Suite, configure o proxy e ative a interceptação de **respostas** (`Intercept responses` habilitado). Realize o login normalmente na plataforma.

```
Proxy → Options → Intercept responses based on the following rules: ✅
```

### Passo 2 — Localizar e modificar o payload

Com a resposta interceptada, localize o campo `plan` no corpo JSON:

```diff
- "plan": "free"
+ "plan": "plus"
```

### Passo 3 — Liberar a resposta manipulada

Encaminhe a resposta modificada ao navegador. O front-end processa o novo valor e desbloqueia as funcionalidades Premium para a sessão ativa.

```
Burp Suite → Forward  →  [Navegador recebe "plan": "plus"]  →  UI Premium desbloqueada
```

### Diagrama do ataque

```
[Cliente]           [Burp Suite / Proxy]          [Servidor]
    |                        |                         |
    |── POST /auth/signin ──>|── POST /auth/signin ───>|
    |                        |<── 200 OK (plan:free) ──|
    |                        |                         |
    |                 [Edita plan:plus]                |
    |                        |                         |
    |<── 200 OK (plan:plus) ─|                         |
    |                        |                         |
[UI Premium ativada]         |                         |
```

---

## 💥 Impacto

**O que foi possível fazer:**
- Acessar ferramentas reservadas ao plano pago (velocidade, isolamento, loop) sem custo.

**O que NÃO foi possível fazer:**
- Persistir o acesso após nova autenticação (sem manipulação).
- Escalar para outros usuários (falha isolada à sessão do atacante).

A falha confirma que o controle de acesso era **puramente cosmético** na camada de UI, sem respaldo de verificação server-side para as funcionalidades afetadas.

---

## ✅ Recomendações

### 1. Validação no servidor (prioridade alta)
Todo controle de acesso a funcionalidades deve ser verificado no back-end a cada requisição, independentemente do que o cliente afirma.

```
❌ Errado:  if (userData.plan === "plus") { enableFeature() }
✅ Correto: Servidor verifica o plano via JWT/DB antes de processar a requisição
```

### 2. Confiar apenas em claims assinadas
O JWT já está presente na resposta. As permissões de plano deveriam ser embutidas como **claims** no próprio token — que é assinado e não pode ser alterado sem invalidação.

```json
// Claim dentro do JWT payload (não mutável pelo cliente)
{
  "sub": "usr_000",
  "plan": "free",
  "exp": 1714000000
}
```

### 3. Nunca confiar em payloads JSON mutáveis para autorização
Dados fora do token assinado são controláveis pelo cliente. Qualquer campo JSON retornado fora do JWT deve ser tratado como **informação de exibição**, nunca como fonte de verdade para permissões.

### 4. Validação de funcionalidades no endpoint
Cada endpoint ou WebSocket que sirva uma funcionalidade Premium deve verificar o plano do usuário diretamente, consultando o banco de dados ou decodificando o JWT com verificação de assinatura.

---

## 📚 Lições Aprendidas

> *"O front-end é território do usuário. Nunca delegue decisões de segurança a ele."*

- **Separação de responsabilidades:** Autenticação (quem é você) e Autorização (o que você pode fazer) são problemas distintos — e ambos pertencem ao servidor.
- **Defense in depth:** A ausência de validação server-side para as features de UI expôs a falha. Downloads permaneceram seguros por terem essa camada adicional.
- **Confiança zero no cliente:** Qualquer dado que trafega entre servidor e cliente deve ser considerado modificável pelo usuário.

---

## 📎 Referências

- [OWASP – Broken Access Control (A01:2021)](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [CWE-602: Client-Side Enforcement of Server-Side Security](https://cwe.mitre.org/data/definitions/602.html)
- [OWASP Testing Guide – Testing for Bypassing Authorization Schema](https://owasp.org/www-project-web-security-testing-guide/)
- [PortSwigger – Modifying serialized objects](https://portswigger.net/web-security/deserialization/exploiting)

---

<div align="center">

*Feito para fins educacionais e de conscientização em segurança.*  
*Pratique ethical hacking apenas em ambientes autorizados.*

</div>
