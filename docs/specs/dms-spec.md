# Especificação - Document Management System

## 1. Objetivo

Permitir que usuários enviem, consultem e baixem seus documentos, armazenados no filesystem local da aplicação.

## 2. Escopo

### Dentro do escopo

- Upload de um documento por requisição.
- Listagem dos documentos pertencentes ao usuário identificado na requisição.
- Download de um documento pelo identificador, respeitando o dono.
- Interface React para upload, listagem e download.
- Identificação simples do usuário para separar documentos nesta fase.

### Fora do escopo

- Autenticação, autorização robusta e gestão de contas.
- Armazenamento externo ou em nuvem.
- Versionamento, edição e exclusão de documentos.
- Persistência de metadados em banco de dados.
- Compartilhamento de documentos entre usuários.
- Restrições de tipo MIME; o limite de tamanho é aplicado a todos os arquivos.

## 3. Requisitos funcionais

| ID | Requisito |
| --- | --- |
| RF-01 | O usuário pode enviar um arquivo por `multipart/form-data`, no campo `file`. |
| RF-02 | O sistema valida a presença do arquivo e o limite máximo configurado. |
| RF-03 | O sistema grava o arquivo no filesystem local com `multer` e `diskStorage`. |
| RF-04 | O sistema registra metadados em memória, incluindo identificador, nome original, tamanho, data e dono. |
| RF-05 | O usuário pode listar somente os documentos associados à sua identificação. |
| RF-06 | O usuário pode baixar um documento pelo identificador se ele pertencer a esse usuário. |
| RF-07 | A interface exibe o resultado do upload, atualiza a listagem e oferece download. |
| RF-08 | A API retorna erros em formato JSON consistente para entradas inválidas, documentos ausentes e falhas de I/O. |

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos devem permanecer no armazenamento local da aplicação, sem provedores externos. |
| RNF-02 | O armazenamento deve usar `multer` com `diskStorage`; o nome físico do arquivo deve ser gerado pelo sistema, não pelo cliente. |
| RNF-03 | Os metadados ficam em memória e são perdidos quando o processo backend reinicia; arquivos locais podem permanecer órfãos após reinício. |
| RNF-04 | O diretório de armazenamento e o limite de upload devem ser configuráveis por variáveis de ambiente. |
| RNF-05 | Arquivos não devem ser servidos diretamente como conteúdo estático; o download passa pelo endpoint e pela verificação de dono. |
| RNF-06 | O backend usa CommonJS, Express e as camadas `routes -> controllers -> services -> repositories`. |
| RNF-07 | O frontend usa React e acessa a API via `fetch` sob o prefixo `/api`, conforme o proxy Vite existente. |
| RNF-08 | A identificação por cabeçalho descrita nesta especificação não é autenticação e não deve ser considerada segura para produção multiusuário. |

## 5. Modelo de dados

### Metadados expostos pela API

| Campo | Tipo | Descrição |
| --- | --- | --- |
| `id` | string | Identificador único gerado pelo sistema. |
| `originalName` | string | Nome original enviado pelo cliente. |
| `size` | number | Tamanho do arquivo em bytes. |
| `uploadedAt` | string | Data/hora UTC em ISO 8601. |
| `owner` | string | Identificação do usuário dono. |

### Dados internos do repositório

Além dos metadados expostos, o registro interno mantém `storageName`, nome gerado para localizar o arquivo no diretório local. Esse valor nunca é retornado ao cliente. O repositório associa cada registro ao caminho construído dentro do diretório configurado e não aceita caminhos enviados pelo cliente.

## 6. Contratos de API

Todas as rotas de documentos exigem o cabeçalho `X-User-Id`, não vazio. Nesta fase ele identifica o usuário para fins de separação funcional, mas pode ser falsificado pelo cliente; autenticação real está fora do escopo. A interface envia esse cabeçalho usando a identidade de demonstração configurada para o ambiente.

### `POST /upload`

- Cabeçalho: `X-User-Id: <identificador>`.
- Entrada: `multipart/form-data` com um arquivo no campo `file`.
- Limite padrão: 10 MiB, configurável por `MAX_FILE_SIZE_BYTES`.
- Sucesso: `201 Created`, JSON com os metadados do documento.
- Erros: `400` para arquivo ausente ou cabeçalho inválido; `413` para arquivo acima do limite; `500` para falha de armazenamento.
- O nome físico é gerado pelo sistema; o nome original é preservado somente como metadado e usado no nome sugerido para download.

### `GET /documents`

- Cabeçalho: `X-User-Id: <identificador>`.
- Sucesso: `200 OK`, JSON `{ "documents": [...] }`, filtrado pelo dono e ordenado do mais recente para o mais antigo.
- Erro: `400` para cabeçalho ausente ou inválido.

### `GET /documents/:id/download`

- Cabeçalho: `X-User-Id: <identificador>`.
- Sucesso: `200 OK`, conteúdo binário com `Content-Disposition: attachment` usando o nome original de forma segura.
- Erro: `404 Not Found` quando o documento não existe, não pertence ao usuário ou o arquivo local não está disponível.
- Erro: `400` para cabeçalho ou identificador inválido.

### Formato dos erros

```json
{
  "error": {
    "code": "DOCUMENT_NOT_FOUND",
    "message": "Documento não encontrado."
  }
}
```

Os códigos e mensagens devem ser estáveis o suficiente para a interface tratar erros sem depender de texto específico. Não retornar caminhos locais, stack traces ou nomes internos de armazenamento.

### Endpoint de saúde existente

`GET /health` continua retornando `200 OK` com `{ "status": "ok" }` e não exige identificação de usuário.

## 7. Decisões arquiteturais

- `routes/`: mapeia endpoints e encaminha requisições aos controllers. Não contém regras de negócio nem acessa filesystem.
- `controllers/`: valida parâmetros HTTP, cabeçalhos e campos; traduz resultados em status e respostas HTTP. Não implementa persistência.
- `services/`: aplica as regras de upload, propriedade, listagem e download. Depende dos contratos do repositório, não de Express ou Multer.
- `repositories/`: grava e lê arquivos locais e mantém os metadados em memória. Isola filesystem e detalhes de persistência do restante do backend.
- Multer usa `diskStorage`; a configuração de destino e a geração do nome físico ficam no limite de infraestrutura.
- O frontend consome a API via `fetch` no prefixo `/api`; o proxy Vite já encaminha esse prefixo ao backend local.
- Não introduzir banco de dados ou armazenamento externo nesta fase.
- A separação por `X-User-Id` é apenas uma convenção de desenvolvimento. Antes de uso com dados reais, deve ser substituída ou protegida por autenticação e identidade confiável.

## 8. Plano de execução

1. **Fechar contratos e configurações.** Confirmar nomes das variáveis, limite de upload e identidade de demonstração. Critério de aceite: contratos e limites documentados, sem implementação de backend ou frontend nesta etapa.
2. **Implementar backend por camadas.** Criar rotas, controllers, services e repositories para upload, listagem e download, mantendo arquivos locais e metadados em memória. Critério de aceite: contratos desta especificação atendidos e testes cobrindo sucesso, validação, propriedade, limites e falhas de arquivo.
3. **Implementar a interface.** Construir fluxo de upload, listagem, estados de carregamento/erro e download, consumindo a API via `/api`. Critério de aceite: usuário consegue concluir os três fluxos e visualizar erros sem recarregar a aplicação.
4. **Validar integração e limitações.** Executar testes backend e build frontend; verificar persistência local dos arquivos e perda de metadados após reinício. Critério de aceite: fluxos integrados funcionam e a limitação de metadados em memória está explícita.

## 9. Configuração inicial

- `PORT`: porta HTTP do backend; padrão atual `3000`.
- `STORAGE_DIR`: diretório local dos arquivos; padrão `backend/storage`.
- `MAX_FILE_SIZE_BYTES`: tamanho máximo permitido; padrão `10485760` bytes.
- Identidade de demonstração: configurada no frontend para enviar `X-User-Id`; não representa autenticação.