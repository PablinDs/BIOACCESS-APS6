  # BIOACCESS
Este projeto foi elaborado como parte da Atividade Prática Supervisionada (APS) do sexto semestre do curso de Ciência da Computação.

# RELATÓRIO TÉCNICO — PROJETO “BIOACCESS”

## 1. Introdução

Este documento tem como objetivo detalhar a estrutura do repositório BIOACESS-APS, descrevendo a função de cada pasta, arquivo e quando pertinente, de cada função relevante implementada. Além disso, busca demonstrar como o sistema cumpre o propósito da atividade: realizar a autenticação de usuários por meio de reconhecimento facial e disponibilizar relatórios de acordo com os níveis hierárquicos de acesso.

## 2. Visão geral do sistema

- **Nome interno:** BioAccess APS  
- **Tecnologias principais:** Python 3.10+, Flask, OpenCV (LBPH), NumPy, HTML/CSS/JavaScript.  
- **Fluxo macro:** cadastro de usuários → treinamento/atualização do modelo → autenticação biométrica via webcam → confirmação de login → acesso a relatórios segmentados por nível (1, 2 ou 3).

## 3. Estrutura de diretórios

| Caminho | Conteúdo | Descrição |
| ------- | -------- | --------- |
| `app.py` | Código-fonte principal | Aplicação Flask com rotas, reconhecimento facial e regras de negócio. |
| `data/` | Dados persistidos | Galeria de rostos, modelo LBPH treinado e metadados dos usuários. |
| `pages/` | Templates Jinja2 | HTML renderizado pelo Flask (index, registro, autenticação, login e dashboard). |
| `static/` | Recursos estáticos | Folhas de estilo e scripts JavaScript usados pelas páginas. |
| `refe/` | Protótipo/Referência | Implementação parcial em React/TypeScript utilizada como referência visual e UX. |
| `requirements.txt` | Dependências Python | Lista mínima para instalar e executar o backend. |
| `GUIA_DO_USUARIO.md` | Manual operacional | Roteiro para usuários finais operarem o sistema. |
| `RELATORIO.md` | Documento atual | Detalhamento técnico do projeto. |
| `APS - 5o e 6o CC 2025_2.pdf` | Documento externo | Enunciado oficial da APS utilizado como requisito. |

## 4. Detalhamento dos componentes

### 4.1 Arquivo `app.py`

Responsável por toda a lógica de servidor, persistência mínima em arquivos e integração com o motor de reconhecimento facial do OpenCV. A tabela abaixo descreve cada elemento relevante:

| Elemento | Tipo | Descrição |
| -------- | ---- | --------- |
| `BASE_DIR`, `DATA_DIR`, `KNOWN_FACES_DIR`, `MODEL_DIR`, `MODEL_PATH`, `LABEL_MAP_PATH`, `USERS_METADATA_PATH` | Constantes | Paths padrão utilizados para armazenar dados da aplicação. |
| `CASCADE_PATH` | Constante | Caminho do classificador Haar Cascade usado para detecção facial. |
| `CONFIDENCE_THRESHOLD`, `PIXEL_DISTANCE_THRESHOLD`, `REQUIRED_CONSISTENT_MATCHES`, `FACE_SIZE`, `LOGIN_MIN_SCORE` | Constantes | Limiar de confiança, verificação pixel-wise, número de confirmações consecutivas, tamanho dos recortes faciais e pontuação mínima para login. |
| `app` | Instância Flask | Configuração principal do servidor, incluindo `SECRET_KEY` e tamanho máximo de upload. |
| `inject_frontend_constants()` | Context processor | Injeta constantes nos templates para uso no front-end. |
| `lbph_model`, `label_index`, `FACE_CASCADE` | Variáveis globais | Modelo LBPH carregado em memória, mapa de rótulos e classificador Haar Cascade pré-carregado. |
| `SENSITIVE_DATA` | Dicionário | Estrutura com os relatórios liberados por nível de acesso (1, 2 e 3). |
| `ensure_directories()` | Função | Garante que diretórios e `users.json` existam. |
| `load_users_metadata()` / `save_users_metadata()` | Funções | Lê/Escreve o arquivo `data/users.json` com informações de usuários e níveis. |
| `detect_largest_face(grayscale)` | Função | Executa `detectMultiScale` e aplica margens para retornar o maior rosto detectado. |
| `extract_face(image)` | Função | Converte a imagem para escala de cinza, detecta o rosto, redimensiona, aplica CLAHE e máscara elíptica; retorna recorte tratado e bounding box. |
| `verify_against_gallery(face_img, username)` | Função | Compara o rosto detectado com amostras salvas do usuário usando distância Euclidiana média. |
| `decode_image(data_url)` | Função | Decodifica imagens base64 enviadas pelo navegador para array NumPy. |
| `collect_training_samples()` | Função | Percorre a galeria e monta listas de amostras/labels para treinamento do LBPH; também cria mapa label → usuário. |
| `calculate_validation_score(confidence)` | Função | Converte a confiança (quanto menor, melhor) em porcentagem amigável para exibição no front-end. |
| `train_model()` | Função | Treina o modelo LBPH com as amostras disponíveis e salva em `data/model`. |
| `load_model_from_disk()` | Função | Carrega modelo e mapa de rótulos do disco quando o servidor inicia. |
| `analyze_frame(image)` | Função | Fluxo completo para processar um frame: extrai rosto, prevê com LBPH, verifica limiares e retorna JSON com bounding box, confiança, usuário e status da verificação extra. |
| `_reset_pending_match()` | Função | Limpa contador de frames consistentes armazenado na sessão. |
| `_update_pending_match(username, confidence)` | Função | Atualiza contador de frames para determinado usuário até atingir `REQUIRED_CONSISTENT_MATCHES`. |
| `index()` | Rota GET `/` | Renderiza página inicial (`pages/index.html`). |
| `register()` | Rota GET/POST `/register` | Gerencia cadastro de usuários, tratamento de fotos e disparo do treinamento do modelo. |
| `auth()` | Rota GET `/auth` | Renderiza a tela de captura e reconhecimento facial. |
| `logout()` | Rota POST `/logout` | Limpa a sessão e redireciona para a tela de autenticação. |
| `api_recognize()` | Rota POST `/api/recognize` | Recebe frames da webcam, chama `analyze_frame()` e controla o fluxo de identificação/pending/erro. |
| `login()` | Rota GET/POST `/login` | Exibe informação do usuário reconhecido, exige confirmação nominal e termo de responsabilidade. |
| `dashboard()` | Rota GET `/dashboard` | Exibe relatórios permitidos ao usuário autenticado. |
| `api_data()` | Rota GET `/api/data` | Endpoint JSON autenticado que retorna os mesmos dados do dashboard. |
| `healthcheck()` | Rota GET `/health` | Retorna status `ok` para monitoramento. |
| bloco `if __name__ == "__main__":` | Inicialização | Garante diretórios, tenta carregar modelo e inicia o servidor em modo debug. |

### 4.2 Diretório `pages/`

Templates em Jinja2/HTML utilizados pelo Flask:

- `base.html`: layout principal, inclui CSS/JS globais e define blocos de conteúdo.
- `index.html`: landing page explicativa com links para registro e autenticação.
- `register.html`: formulário de cadastro; integra scripts de upload e captura.
- `auth.html`: tela de reconhecimento em tempo real, incluindo canvas e feedback.
- `login.html`: etapa de confirmação do usuário reconhecido, exibe pontuação e termo de responsabilidade.
- `dashboard.html`: lista relatórios de acordo com o nível, mostra dados da sessão.

### 4.3 Diretório `static/`

#### 4.3.1 `static/css/`

- `styles.css`: estilos base (tipografia, layout, cards, botões, feedbacks visuais do auth).
- `login.css`: estilos adicionais para a tela de login (dark mode, barra de confiança, componentes reutilizáveis).

#### 4.3.2 `static/js/`

- `auth.js`: módulo IIFE que cuida da captura de vídeo, sincronização de canvas, envio periódico de frames para `/api/recognize`, interpretação da resposta e feedback visual (bounding box, mensagens, controle de fluxo).
- `register.js`: gerencia câmera da página de cadastro, galeria de capturas, contadores, remoção de imagens e montagem do campo oculto `captured_photos`.
- `login.js`: aplica preferência de tema (claro/escuro) e anima a barra de pontuação exibida no login.

Os scripts são carregados nos templates correspondentes por meio dos blocos definidos em `base.html`.

### 4.4 Diretório `data/`

- `known_faces/`: contém subpastas por usuário (ex.: `Pablo Victor/`) com arquivos `face_XXX.png` gerados durante o cadastro.
- `model/lbph_model.xml`: modelo LBPH persistido após treinamento.
- `model/label_map.json`: dicionário que mapeia índices numéricos do modelo para nome de usuário e nível de acesso.
- `users.json`: metadados dos usuários (nome como chave, nível como valor).

> Todos os dados são guardados localmente, sem banco de dados relacional, para facilitar a entrega acadêmica.

### 4.5 Diretório `refe/`

Reúne um protótipo em React/TypeScript (`Login.tsx` + `Login.css`) que demonstra uma interface alternativa com múltiplos modos de login (tradicional, upload de imagem, câmera). Serve como referência de design e não é utilizado diretamente pela aplicação Flask.

- `Login.tsx`: componente principal com hooks de estado, integração a contextos de tema e autenticação, ações de upload e captura.
- `Login.css`: estilos associados ao protótipo React.

### 4.6 Outros arquivos

- `requirements.txt`: lista dependências essenciais (`flask`, `opencv-contrib-python`, `numpy` etc.).
- `APS - 5o e 6o CC 2025_2.pdf`: enunciado que descreve objetivos da APS (controle biométrico + relatórios por nível).
- `GUIA_DO_USUARIO.md`: passo a passo simplificado para cadastro, reconhecimento e demonstração do sistema.

## 5. Funcionamento detalhado

1. **Cadastro**: fotos enviadas são pré-processadas (`extract_face`) para uniformizar iluminação e remover fundo. Os arquivos são numerados e salvos. Ao final, `train_model()` atualiza o modelo LBPH e grava `label_map.json`.
2. **Reconhecimento**: a cada frame recebido via `/api/recognize`, o sistema tenta detectar o rosto (`extract_face`), prediz com LBPH (`lbph_model.predict`), verifica limiar de confiança e, se aprovado, faz checagem extra `verify_against_gallery`. O estado de pendência é mantido em sessão com `_update_pending_match`.
3. **Login controlado**: somente após 5 acertos consecutivos e pontuação mínima (`LOGIN_MIN_SCORE`) a sessão recebe o usuário e a taxa de validação; o formulário `/login` exige confirmação manual antes de liberar o dashboard.
4. **Autorização por nível**: `dashboard()` e `api_data()` compõem a lista de relatórios, agregando `SENSITIVE_DATA` de 1 até o nível do usuário autenticado.

## 6. Considerações sobre manutenção

- **Limpeza do modelo**: para reiniciar do zero, apagar `data/model/*`, `data/known_faces/*` e `data/users.json`. O aplicativo recriará a estrutura quando reiniciar.
- **Ajustes de limiares**: modificar as constantes no topo de `app.py` conforme o ambiente de testes.
- **Evoluções possíveis**: integração com banco de dados, adoção de pipelines modernos de reconhecimento (CNN), deploy em nuvem com HTTPS e logs centralizados.

## 7. Conclusão

O repositório oferece uma solução completa alinhada aos objetivos acadêmicos da APS: captura, tratamento e reconhecimento facial com controle de acesso por níveis, apoiado por documentação. A modularização facilita futuras melhorias, tanto no motor biométrico quanto na camada de apresentação.
