# Corretiza

**Atendimento imobiliário automatizado no WhatsApp.**
O cliente pergunta "tem apartamento de 2 quartos para alugar no centro?" e recebe os imóveis que
batem com o pedido — buscados no catálogo da própria imobiliária, formatados e enviados na conversa.

É uma plataforma multi-empresa: cada imobiliária tem suas instâncias de WhatsApp, seus agentes de
IA, seu catálogo de imóveis, seus corretores e seus leads, isolados das demais.

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat-square&logo=nodedotjs&logoColor=white)
![Express](https://img.shields.io/badge/Express-000000?style=flat-square&logo=express&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white)
![Drizzle](https://img.shields.io/badge/Drizzle%20ORM-C5F74F?style=flat-square&logo=drizzle&logoColor=black)
![React](https://img.shields.io/badge/React-61DAFB?style=flat-square&logo=react&logoColor=black)
![Tailwind](https://img.shields.io/badge/Tailwind-06B6D4?style=flat-square&logo=tailwindcss&logoColor=white)
![Socket.IO](https://img.shields.io/badge/Socket.IO-010101?style=flat-square&logo=socketdotio&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-412991?style=flat-square&logo=openai&logoColor=white)
![WhatsApp](https://img.shields.io/badge/WhatsApp-25D366?style=flat-square&logo=whatsapp&logoColor=white)

---

## O que ele faz

**Agente de IA que conhece o catálogo**
O agente entende o pedido em linguagem natural, filtra o catálogo por tipo de imóvel, transação,
quartos, banheiros, vagas, cidade, bairro e área, e devolve os imóveis formatados com endereço,
características, descrição e link de localização.

**Time de agentes, não um agente só**
Cada instância de WhatsApp tem um agente principal e pode ter agentes secundários especializados.
A conversa é delegada ao especialista certo por palavra-chave — locação, venda e financiamento
podem ter tom, prompt e material de treinamento diferentes.

**Treinamento por documento**
O material de treinamento do agente pode ser carregado em PDF, extraído e incorporado ao contexto
— a imobiliária ensina o agente com o material que já tem.

**Operação comercial completa**
Painel de conversas em tempo real, gestão de leads e atendimentos, cadastro de corretores, imóveis,
cidades e comodidades, agenda de visitas, disparo em massa e listas de transmissão.

**Administração da plataforma**
Área separada para gestão de empresas, usuários, planos, configurações globais de IA e das
instâncias de WhatsApp, com geração de QR Code para conectar cada número.

---

## Decisões técnicas que valem nota

**O agente espera o cliente terminar de falar.**
Ninguém escreve tudo numa mensagem só — manda "boa tarde", depois "tudo bem?", depois o que quer
de fato. Um agregador guarda as mensagens num buffer por número e por instância e reinicia um timer
de 15 segundos a cada nova mensagem; quando o cliente para, tudo vira um texto único e o agente
responde uma vez. Sem isso, o bot responde três vezes e a conversa fica sem pé nem cabeça.

**Mídia fura a fila do agregador.**
Imagem e áudio são processados na hora, sem esperar o timer — quem manda uma foto espera resposta
imediata, e áudio não se concatena com texto.

**Filtro determinístico antes do modelo.**
A intenção de busca é detectada por vocabulário e os critérios numéricos — quartos, banheiros,
vagas — saem por expressão regular. Só o que sobra depende do modelo. Mais barato, mais previsível,
e "3 quartos" nunca vira "2 quartos" por alucinação.

**Isolamento por empresa em toda a cadeia.**
Usuários pertencem a uma empresa, e instâncias de WhatsApp, agentes, conversas e imóveis são
escopados por ela. Um middleware dedicado impõe a fronteira antes do handler, em vez de deixar cada
rota lembrar de filtrar.

**Inicialização tardia da camada de dados.**
A conexão com o banco é criada sob demanda, e não na importação do módulo — garante que as
variáveis de ambiente já estejam carregadas quando a conexão for aberta, um erro clássico de ordem
de import que só aparece em produção.

**Número de telefone normalizado na entrada.**
O identificador do remetente chega em formatos diferentes conforme o tipo de conta do WhatsApp;
a extração normaliza os dois antes de qualquer coisa, porque é essa chave que amarra conversa,
buffer e lead.

**Resultado limitado por desenho.**
A busca devolve no máximo cinco imóveis. Não é limite técnico: mandar quinze no WhatsApp faz o
cliente parar de ler.

---

## Como está construído

```
Mensagem no WhatsApp
   └── webhook da API de mensageria
        └── agregador: mídia passa direto · texto espera 15s pela próxima
             └── resolve empresa, instância e agente
                  └── delegação por palavra-chave → agente especializado
                       ├── intenção de busca? → filtros → catálogo da empresa
                       └── senão → modelo, com prompt e treinamento do agente
                            └── resposta enviada e conversa persistida
```

Backend em Express com TypeScript, frontend em React servido pelo mesmo processo, banco MySQL
com Drizzle, tempo real por Socket.IO e armazenamento de objetos para as mídias.

---

## Rodando localmente

```bash
npm install
npm run db:push     # aplica o schema no banco
npm run dev         # servidor com recarga automática
```

Produção: `npm run build` (Vite para o cliente, esbuild para o servidor) e `npm start`.
Verificação de tipos com `npm run check`.

---

<details>
<summary><b>Referência — ambiente, estrutura e integrações</b></summary>

### Variáveis de ambiente

| Variável | Para que serve |
|---|---|
| `MYSQL_HOST` · `MYSQL_USER` · `MYSQL_PASSWORD` · `MYSQL_DATABASE` | Conexão com o banco |
| `JWT_SECRET` | Assinatura do token de sessão |
| `NODE_ENV` | Ambiente de execução |

As credenciais da API de mensageria e do provedor de IA são configuradas pela área administrativa,
não por arquivo.

### Estrutura

```
├── client            SPA em React + Vite
│   └── src/pages
│       ├── admin     empresas, usuários, planos, configurações, instâncias
│       └── client    agentes, conversas, imóveis, corretores, leads, agenda, disparos
├── server
│   ├── services      mensageria, webhook, agregador, IA, imóveis, e-mail, agendamento
│   ├── storage.ts    camada de dados com escopo por empresa
│   ├── auth.ts       autenticação e papéis
│   └── routes.ts     rotas da API
├── shared            schema e tipos compartilhados entre cliente e servidor
└── migrations        migrations geradas pelo ORM
```

### Integrações

- **Mensageria** — criação e gestão de instâncias, QR Code de conexão, configuração de webhook e
  preferências como leitura automática e rejeição de chamadas
- **IA** — geração de texto, análise de imagem e transcrição de áudio
- **Armazenamento de objetos** — mídias e avatares, com controle de acesso próprio
- **E-mail** — notificações transacionais

</details>
