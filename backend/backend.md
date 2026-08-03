# SoundWeaves - Especificação Técnica e Regras de Negócio

## 1. Modelagem de Dados (Tabelas do Banco)

### 1.1. `User` (Extensão do Auth Django)
O Django já gerencia senhas com hash, e-mails e permissões. Estenderemos o modelo padrão (`AbstractUser`) para adicionar campos extras.
- `id` (PK, UUID ou AutoField)
- `username`, `email`, `password` (Padrão Django)
- `birth_date` (Date, null=True)
- **Roles (Nativas do Django)**:
  - `is_superuser` (Booleano) -> **Admin**
  - `is_staff` (Booleano) -> **Staff / Moderador**
  - `is_active` (Booleano) -> Usado para banir/suspender (se False, o usuário não loga)

### 1.2. `Category` (Categorias de Áudio)
- `id` (PK)
- `name` (String, max=100, unique)
- `description` (Text, null=True)
- `created_at` (DateTime)

### 1.3. `Track` (Música / Faixa de Áudio)
Esta tabela guarda os metadados. Os arquivos reais vão para o S3.
- `id` (PK, UUID recomendado para URLs seguras)
- `title` (String, max=255)
- `artist` (FK -> User) -> Criador/Dono da faixa
- `category` (FK -> Category) -> Relacionamento 1:N
- `audio_file` (FileField) -> O Django/Boto3 salva o caminho gerado no S3
- `cover_image` (ImageField, null=True) -> Caminho no S3
- `lyrics` (TextField, null=True)
- `description` (TextField, null=True)
- `visibility` (Enum/Choices):
  - `PUBLIC` (Qualquer pessoa)
  - `LOGGED_IN` (Apenas usuários autenticados)
  - `PRIVATE` (Apenas o próprio criador)
- `status` (Enum/Choices):
  - `PENDING` (Aguardando aprovação)
  - `APPROVED` (Pública no catálogo)
  - `REJECTED` (Recusada pelo moderador)
- `play_count` (Integer, default=0) -> Cache total de reproduções
- `created_at` (DateTime)
- `updated_at` (DateTime)

### 1.4. `Playlist` (Playlists dos Usuários)
- `id` (PK)
- `title` (String, max=255)
- `user` (FK -> User) -> Dono da playlist
- `tracks` (ManyToMany -> Track) -> Uma playlist tem várias músicas, e uma música pode estar em várias playlists.
- `created_at` (DateTime)

### 1.5. `TrackLike` (Curtidas / Músicas Salvas)
Tabela pivô para registrar quais músicas o usuário curtiu/salvou.
- `id` (PK)
- `user` (FK -> User)
- `track` (FK -> Track)
- `created_at` (DateTime)
*(Regra: UniqueConstraint(user, track) para não curtir duas vezes)*

### 1.6. `TrackPlayHistory` (Histórico de Visualizações/Plays) - *Opcional mas recomendado*
Para registrar *quem* ouviu *quando* (bom para análises futuras e evitar fraudes de "plays").
- `id` (PK)
- `track` (FK -> Track)
- `user` (FK -> User, null=True) -> Null se for usuário anônimo
- `played_at` (DateTime)


---

## 2. Regras de Negócio e Permissões (Roles)

O sistema conta com 4 níveis de acesso principais. Toda requisição (API ou View) deve respeitar estas travas:

### 2.1. Usuário Anônimo (Não Logado)
- **Pode:**
  - Navegar pelo site e explorar categorias.
  - Reproduzir músicas que tenham `visibility = PUBLIC` E `status = APPROVED`.
- **Não Pode:**
  - Curtir/Salvar músicas, criar playlists, fazer upload de faixas.
  - Ver músicas marcadas como `LOGGED_IN` ou `PRIVATE`.

### 2.2. Usuário Autenticado (Comum)
- **Pode:**
  - Fazer tudo do Usuário Anônimo.
  - Ouvir músicas com `visibility = LOGGED_IN` (desde que estejam `APPROVED`).
  - Fazer **Upload** de novas músicas (`Track`) — que entrarão no status `PENDING` automaticamente.
  - Editar e Excluir **apenas as próprias** músicas.
  - Criar, editar e excluir **apenas as próprias** Playlists.
  - Dar Like / Salvar qualquer música permitida a ele.
- **Não Pode:**
  - Acessar o Painel Admin do Django.
  - Criar categorias.

### 2.3. Staff / Moderador (`is_staff = True`)
- **Pode:**
  - Fazer tudo do Usuário Comum.
  - Acessar o **Django Admin** (com permissões limitadas).
  - Ver todas as músicas no status `PENDING`.
  - **Aprovar** (`APPROVED`) ou **Remover/Recusar** (`REJECTED`) faixas do catálogo.
  - **Suspender usuários** comuns (mudando a flag `is_active` para `False`).
- **Não Pode:**
  - Hard Delete (apagar registros do banco permanentemente, usar apenas Soft Delete ou inativar).
  - Suspender outros Staffs ou Admins.
  - Criar ou editar Categorias.

### 2.4. Admin / Superusuário (`is_superuser = True`)
- **Pode:**
  - Fazer **TUDO** (Acesso total ao Django Admin).
  - Criar, editar e excluir **Categorias**.
  - Banir/Suspender qualquer usuário (incluindo Staffs).
  - Promover usuários a Staff.
  - Realizar exclusão física (Hard Delete) de registros se necessário.