Usando submódulos do git em pipelines do Azure DevOps

Este post explica como configurar o uso de git submodules em pipelines de CI/CD no Azure DevOps.

Em específico, como configurar quando os **submódulos fazem parte do mesmo projeto**.

A solução envolve habilitar o "Checkout submodules" pelo painel do Azure, utilizar URLs relativas no arquivo .gitmodules e adicionar uma chave "resources" no arquivo .yaml da sua pipeline.

<hr/>

## Estrutura do projeto

Você tem um projeto no Azure chamado "Meu App Foda".

Esse projeto tem vários repos:

- frontend\_app (URL: https://dev.azure.com/Minha-Organization/Meu%20App%20Foda/\_git/frontend\_app)
- backend\_app (URL: https://dev.azure.com/Minha-Organization/Meu%20App%20Foda/\_git/backend\_app)
- frontend\_dependencies (URL: https://dev.azure.com/Minha-Organization/Meu%20App%20Foda/\_git/frontend\_dependencies)

E você está usando o "frontend\_dependencies" como um submódulo dentro de "frontend\_app".

A estrutura do "frontend\_app" é:

        - frontend_app
        - frontend_app/.git
        - frontend_app/.gitmodules
        - frontend_app/src
        - frontend_app/package.json
        - frontend_app/package-lock.json
        - frontend_app/frontend_dependencies

Isso quer dizer que você iniciou o "frontend\_dependencies" como um submódulo, com esse comando:

        # dentro de frontend_app
        git submodule add git@ssh.dev.azure.com:v3/Minha-Organization/Meu%20App%20Foda/frontend_dependencies frontend_dependencies

E você possui um **pipeline** configurado no Azure, com o seguinte arquivo .yaml:

        pool:
          vmImage: ubuntu-latest
        steps:
        - script: npm install
          displayName: 'Installs frontend packages'

        - script: npm test
          displayName: 'Run frontend test suite'

Mas existe um problema: pra conseguir rodar os testes, você precisa do submódulo.

Se você tentar rodar o `npm install` sem ele, vai cair em um erro do tipo (lá nos logs da sua pipeline):

        Generating script.
        Script contents:
        npm install
        ========================== Starting Command Output ===========================
        /usr/bin/bash --noprofile --norc /home/vsts/work/_temp/d91f7be5-82d1-4241-bfbe-ab849ccbd3c6.sh
        npm ERR! code ENOENT
        npm ERR! syscall open
        npm ERR! path /home/vsts/work/1/s/frontend_app/frontend_dependencies/package.json
        npm ERR! errno -2
        npm ERR! enoent ENOENT: no such file or directory, open '/home/vsts/work/1/s/frontend_app/frontend_dependencies/package.json'
        npm ERR! enoent This is related to npm not being able to find a file.
        npm ERR! enoent

        ##[error]Bash exited with code '254'.

## Iniciando os submodulos na pipeline

A configuração acontece em três passos:

1. Habilitar o "Checkout submodules" pelo painel do Azure.
2. Utilizar URLs relativas no arquivo .gitmodules.
3. Adicionar uma chave "resources" no arquivo .yaml da sua pipeline.

### Habilitar o "Checkout submodules" pelo painel do Azure

Você precisa acessar a aba "⚡ Triggers" dentro da tela de edição do pipeline.

Vá em pipelines e acesse o pipeline desejado, clique em "Edit", e, no canto superior direito, clique no Kebab (...) e "⚡ Triggers".

Ali, selecione a aba "YAML" (você vai estar em "Triggers"), clique em "Get sources", e, na tela da direita, selecione "Azure Repos Git" (esse passo a passo é para repositórios que estão no mesmo projeto do Azure).

Em "Repository" selecione o repositório que precisa utilizar o submódulo (frontend\_app), e **marque a opção "Checkout Submodules"**.

No dropdown logo abaixo, escolha se você precisa iniciar o submódulo de forma recursiva (se o seu submódulo carrega outros submódulos...) ou apenas carregá-lo ("Top level submodules only").

Agora o seu pipeline vai tentar iniciar todos os submódulos listados no seu .gitmodules, sempre!

### Utilizar URLs relativas no arquivo .gitmodules

Se você concluiu o passo anterior, vai receber um erro de permissão: de dentro da VM do Azure, você não vai ter acesso aos seus repositórios.

        Please make sure you have the correct access rights
        and the repository exists.
        fatal: clone of 'git@ssh.dev.azure.com:v3/Minha-Organization/Meu%20App%20Foda/frontend_dependencies' into submodule path '/home/vsts/work/1/s/frontend_app/frontend_dependencies' failed
        Failed to clone 'frontend_app/frontend_dependencies'. Retry scheduled
        Cloning into '/home/vsts/work/1/s/frontend_app/frontend_dependencies'...
        Permission denied, please try again.
        Permission denied, please try again.
        git@ssh.dev.azure.com: Permission denied (password,publickey).
        fatal: Could not read from remote repository.

Precisamos alterar a URL dos submodulos de URL absoluta para URL relativa.

Edite o arquivo `frontend_app/.gitmodules`

        [submodule "frontend_dependencies"]
            path = frontend_dependencies
         -  url = git@ssh.dev.azure.com:v3/Minha-Organization/Meu%20App%20Foda/frontend_dependencies
         +  url = ../frontend_dependencies
            branch = main

Como os dois repos estão no mesmo projeto, podemos usar a sintaxe "../".

### Adicionar uma chave "resources" no arquivo .yaml da sua pipeline

Após ajustar as URLs, sua pipeline pode imediatamente funcionar (nesse caso, **CONGRATS**!!), ou você pode receber o seguinte erro:


        Submodule 'frontend_dependencies' (https://dev.azure.com/Minha-Organization/Meu%20App%20Foda/_git/frontend_dependencies) registered for path 'frontend_dependencies'
        Cloning into '/home/vsts/work/1/s/frontend_app/frontend_dependencies'...
        remote: TF401019: The Git repository with name or identifier frontend_dependencies does not exist or you do not have permissions for the operation you are attempting.
        fatal: repository 'https://dev.azure.com/Minha-Organization/Meu%20App%20Foda/_git/frontend_dependencies' not found
        fatal: clone of 'https://dev.azure.com/Minha-Organization/Meu%20App%20Foda/_git/frontend_dependencies' into submodule path '/home/vsts/work/1/s/frontend_app/frontend_dependencies' failed
        Failed to clone 'frontend_dependencies'. Retry scheduled

O Erro **TF401019** significa que seu projeto bloqueia o acesso entre repositórios. Nesse caso, você precisa listar explicitamente quais repositórios sua pipeline pode acessar.

No seu arquivo .yaml, adicione a seguinte chave:

      + resources:
      +   repositories:
      +     - repository: frontend_dependencies # nome do repositório usado como submódulo
      +       type: git
      +       name: frontend_dependencies # use o mesmo que acima
      +       branch: main # defina o branch onde os arquivos que você precisa estão
      +
        pool:
          vmImage: ubuntu-latest
        steps:
        - script: npm install
          displayName: 'Installs frontend packages'

        - script: npm test
          displayName: 'Run frontend test suite'

### Conclusão e referências

Agora sua pipeline consegue acessar submódulos do mesmo projeto do Azure.

Alguns pontos negativos dessa solução são que você sempre vai carregar todos os submodulos no pipeline, mesmo que seu pipeline precise de apenas alguns deles, e que você precisa listar todos os repositórios dentro do arquivo .yaml, assim como ajustar suas urls no .gitmodule.

Referências:

- <https://www.timschaeps.be/post/what-you-should-not-forget-when-running-azure-pipelines-with-git-and-submodules/>
- <https://www.timschaeps.be/post/dealing-with-error-tf401019-submodules-azure-pipelines/>
- <https://developercommunity.visualstudio.com/t/unable-to-clone-private-github-submodule-repositor/430126>
- <https://stackoverflow.com/a/64999223>
- <https://stackoverflow.com/a/56095467>
- <https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/pipeline-options-for-git?view=azure-devops&tabs=yaml#checkout-submodules>
- <https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/steps-checkout?view=azure-pipelines>
- <https://www.damirscorner.com/blog/posts/20210423-ChangingUrlsOfGitSubmodules.html>

Tags: azure, git, ci
