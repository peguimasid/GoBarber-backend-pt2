
# Continuando API do GoBarber

## Aula 18 - Configurando Multer

### Upload de imagem

Usuário seleciona a imagem, o upload já é feito e o servidor retorna o ID da imagem.
E no json no cadastro de usuário por exemplo, envia o ID da imagem.

Utilizando o [multer](https://github.com/expressjs/multer) para upload de arquivos.

Quando precisa enviar imagem para o servidor, tem que ser como `Multpart-data` (Multpart Form) e não `json`.

Instalando o `multer`: 

```
yarn add multer
```

Criar uma pasta fora do `src`, para armazenar as imagens: `tmp/uploads`, dentro da pasta `tmp` criar outra pasta `uploads`, onde vai ficar os arquivos físicos de uploads de arquivos.

Criar um arquivo de configuração `multer.js` de dentro da `config`.

```
import multer from 'multer';
import crypto from 'crypto';
import { extname, resolve } from 'path';

export default {
  storage: multer.diskStorage({
	// Local onde o arquivo será salvo na máquina do servidor
    destination: resolve(__dirname, '..', '..', 'tmp', 'uploads'),
    // Gerando o nome da imagem como um hash usando a lib nativa do node: crypto
    filename: (req, file, cb) => {
      crypto.randomBytes(16, (err, res) => {
        if (err) return cb(err);
        return cb(null, res.toString('hex') + extname(file.originalname));
      });
    },
  }),
};
```

Depois criar um rota:

```
import multer from  'multer';
const upload =  multer(multerConfig);

const upload =  multer(multerConfig);

routes.post('/files', upload.single('file'), (req, res) => {
	return res.json({ ok:  true });
});
```

A rota tem que usar o método post, e o corpo da requisição tem que ser um `multpart-form` em vez de `json`.

Depois adicionar um atributo `file` e adicionar o arquivo nesse atributo.

`upload.single('file')` significa que vou enviar apenas um arquivo dentro da propriedade `file`. 

Essa lib multer permite envio de multiplos arquivos.

Fim: [https://github.com/tgmarinho/gobarber/tree/aula18](https://github.com/tgmarinho/gobarber/tree/aula18)

## Aula 19 - Avatar do Usuário

### Salvando informações do arquivo na base de dados

O atríbuto `req`tem disponível a variável `file` do upload de arquivos que armazena algumas informações sobre o arquivo de upload:

```
{
  "fieldname": "file",
  "originalname": "code-hoc.png",
  "encoding": "7bit",
  "mimetype": "image/png",
  "destination": "/Users/xxx/Developer/bootcamp_rocketseat_studies/gobarber/tmp/uploads",
  "filename": "1d05508938b533ef539026149c597ed5.png",
  "path": "/Users/xxx/Developer/bootcamp_rocketseat_studies/gobarber/tmp/uploads/1d05508938b533ef539026149c597ed5.png",
  "size": 115050
}
```

originalname: é o nome original do arquivo que estava na máquina cliente, que fez o upload.
filename: é o novo nome da imagem que vai ficar salva no servidor.

Para lidar melhor com o upload de arquivo, vou criar o `FileController.js` que conterá a lógica para salvar no banco de dados as referências dos arquivos de upload.

Para salvar os dados do arquivo, vamos criar a tabela files no banco de dados, criando o arquivo de migration.

```
yarn sequelize migration:create --name=create-files
```

E terminar de configurar:

```
module.exports = {
  up: (queryInterface, Sequelize) => {
    return queryInterface.createTable('files', {
      id: {
        type: Sequelize.INTEGER,
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
      },
      name: {
        type: Sequelize.STRING,
        allowNull: false,
      },
      path: {
        type: Sequelize.STRING,
        allowNull: false,
        unique: true,
      },
      created_at: {
        type: Sequelize.DATE,
        allowNull: false,
      },
      updated_at: {
        type: Sequelize.DATE,
        allowNull: false,
      },
    });
  },

  down: queryInterface => {
    return queryInterface.dropTable('files');
  },
};
```

E para gerar a tabela files no banco de dados conforme a migration, só executar no terminal:

```
yarn sequelize db:migrate  
```

Depois criar o Model File:

```
import Sequelize, { Model } from 'sequelize';

class File extends Model {
  static init(sequelize) {
    super.init(
      {
        name: Sequelize.STRING,
        path: Sequelize.STRING,
      },
      {
        sequelize,
      }
    );

    return this;
  }
}

export default File;
```

E inserir o Model File no arquivo database/index.js 


```
...
import File from  '../app/models/File';
...
const models = [User, File];
...
```

E agora atualizar o FileController.js para poder receber os dados da requisição do arquivo de upload, e salvar no banco de dados as referências:

```
import File from '../models/File';

class FileController {
  async store(req, res) {
    const { originalname: name, filename: path } = req.file;

    const file = await File.create({ name, path });

    return res.json(file);
  }
}

export default new FileController();
```

Agora na hora que enviar novamente o arquivo, a tabela dados irá ser preenchida.


### Relacionando o usuário com imagem de avatar (user <-> files)

Para fazer o relacionamento precisamsos adicionar as chaves primária de files no users.

Para isso teremos que criar uma migration para atualizar essas tabelas:

```
yarn sequelize migration:create --name=add-avatar-field-to-users
```

Adicionamos a coluna `avatar_id` de dentro da tabela  `users`, sendo referenciadas pela tabela `files` no atributo `ID` que é a chave primária da tabela `files`. E quando desfazer a migration é só apagar o atributo `avatar_id` de `users`.

`onUpdate: 'CASCADE'`: Quando atualiza a imagem, altera no usuário
`onDelete: 'SET NULL'`: Quando deletar o avatar deixa o avatar_id como null

```
module.exports = {
  up: (queryInterface, Sequelize) => {
    return queryInterface.addColumn('users', 'avatar_id', {
      type: Sequelize.INTEGER,
      references: { model: 'files', key: 'id' },
      onUpdate: 'CASCADE',
      onDelete: 'SET NULL',
      allowNull: true,
    });
  },

  down: queryInterface => {
    return queryInterface.removeColumn('users', 'avatar_id');
  },
};
```

Só executar: `yarn sequelize db:migrate` para executar a alteração na tabela `users`.

Depois precisa relacionar o Users com Files de dentro do Model de users no código.

Adicionando um método para associar as duas entidades:

`Users.js`:
```
...
static associate(models) {
	this.belongsTo(models.File, { foreignKey: 'avatar_id' });
}
...
```

E dentro do `database/index.js`, acresento mais um `map`, para poder executar (apenas nas classes que contém o método associate) a associação e passar os models para o associate:

```
 models
      .map(model => model.init(this.connection))
      .map(model => model.associate && model.associate(this.connection.models));
```

Pronto, agora sim, na hora de salvar o usuário a associação irá ocorrer e o ID que for informado no files estará no users.