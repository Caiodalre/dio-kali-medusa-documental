# Auditoria de senhas com Kali Linux e Medusa

Projeto documental desenvolvido para o desafio da DIO sobre ataques de força bruta e medidas de prevenção em FTP, aplicações web e SMB.

> **Modalidade: estudo documental.** Este repositório apresenta conceitos, planejamento de laboratório e exemplos de comandos. Não foram executados testes durante a elaboração deste material. Não há acessos obtidos, vulnerabilidades comprovadas ou capturas de execução. Os exemplos não representam resultados observados.

## 1. Objetivo

Compreender como senhas previsíveis, ausência de limitação de tentativas e exposição de serviços podem favorecer ataques de autenticação. Relacionar cada cenário com controles de prevenção, detecção e validação.

A entrega utiliza a modalidade documental permitida pelo enunciado. A execução prática é uma possível evolução do projeto.

## 2. Conceitos estudados

| Conceito | Descrição | Exemplo conceitual |
|---|---|---|
| Força bruta | Tentativa sistemática de combinações de credenciais | Experimentar possíveis senhas de uma conta |
| Ataque de dicionário | Tentativas com uma lista limitada de palavras candidatas | Usar uma wordlist curta de laboratório |
| Password spraying | Uma mesma senha candidata testada em várias contas em uma rodada | Uma tentativa por conta com uma única senha |
| Enumeração de usuários | Identificação de possíveis contas existentes | Consultar informações disponibilizadas pelo SMB no laboratório |

Uma wordlist curta não esgota todas as senhas possíveis. Um teste sem sucesso não prova que um serviço é seguro. Spraying também pode causar bloqueios, e sua detecção exige observar várias contas, não apenas uma conta isoladamente.

## 3. Ambiente proposto

| Componente | Função | Endereço ilustrativo |
|---|---|---|
| Kali Linux | Estação de auditoria com Medusa e ferramentas auxiliares | 192.168.56.10 |
| Metasploitable 2 | Alvo deliberadamente vulnerável | 192.168.56.20 |
| DVWA | Aplicação web didática no alvo, quando disponível | http://192.168.56.20/dvwa/ |

Os endereços são exemplos, não foram descobertos na rede. O roteiro considera somente VMs próprias e isoladas. Nenhum endereço externo faz parte do escopo.

### Configuração planejada no VirtualBox

1. Importar ou instalar as duas VMs e criar snapshots antes dos exercícios.
2. Conectar ambas à mesma rede **host-only**, sem adaptador em modo bridge no alvo e sem encaminhamento de portas.
3. Confirmar os endereços atribuídos e verificar se estão na mesma sub-rede.
4. Manter o alvo sem NAT, sem gateway para a internet e sem roteamento pelo host. Instalar previamente as ferramentas no Kali, desconectando seu adaptador adicional antes do exercício.
5. Desativar pastas compartilhadas desnecessárias e registrar as versões utilizadas.

**Precisão de termos:** host-only permite comunicação entre o host e as VMs daquela rede. O modo **Rede Interna** conecta as VMs entre si, sem incluir o host por padrão. São alternativas distintas; aqui foi escolhido host-only para seguir a proposta do desafio. O isolamento também depende de não haver roteamento adicional configurado.

A disponibilidade de FTP, SMB e DVWA deve ser verificada na imagem instalada. Não se presume que versões atuais das ferramentas sejam totalmente compatíveis com serviços antigos do Metasploitable 2.

## 4. Preparação e reconhecimento

Todos os comandos a seguir são **exemplos não executados**, destinados exclusivamente ao alvo do laboratório. Executar a partir da raiz deste repositório no Kali.

```bash
mkdir -p evidencias
ip -br addr
nmap -sV -p 21,80,139,445 192.168.56.20 -oN evidencias/servicos.txt
medusa -V
medusa -d
medusa -M ftp -q
medusa -M smbnt -q
```

O reconhecimento proposto limita a consulta às portas relevantes. Uma porta aberta indica um serviço acessível, não uma autenticação vulnerável. Guardar versão, horário, IP e configuração é necessário para interpretar um teste futuro.

A lista e a ajuda dos módulos devem ser verificadas na instalação local. A ausência de um módulo ou uma incompatibilidade de protocolo é uma limitação técnica, não uma senha incorreta.

## 5. Cenário FTP: ataque de dicionário

**Hipótese:** uma conta didática com senha previsível poderia ser identificada por uma lista pequena, caso o serviço aceite tentativas sucessivas.

O arquivo `wordlists/senhas-lab.txt` contém apenas três candidatos fictícios. O usuário `aluno` é ilustrativo e precisa existir no laboratório para que o exemplo faça sentido.

```bash
medusa -h 192.168.56.20 -u aluno -P wordlists/senhas-lab.txt -M ftp -t 1 -f -O evidencias/ftp.txt
```

O exemplo limita a concorrência a uma autenticação e solicita parada após encontrar uma credencial válida. Isso não implementa um intervalo mínimo entre todas as tentativas.

**Validação planejada:** se a ferramenta indicar sucesso, fazer um único login manual com o cliente FTP, confirmar a identidade com os recursos disponíveis e encerrar a sessão. Não enviar, remover ou alterar arquivos. Verificar se houve login autenticado em vez de acesso anônimo.

**Limitações:** conexão recusada, indisponibilidade, bloqueio e erro de protocolo devem ser registrados separadamente de rejeição de credenciais.

**Prevenção:** substituir FTP sem criptografia por SFTP quando aplicável, usar senhas longas e exclusivas, restringir a rede de acesso e monitorar falhas. SFTP protege o transporte; sozinho não impede tentativas de adivinhação.

## 6. Cenário web: automação no DVWA

**Hipótese:** um formulário sem limitação de tentativas pode permitir repetição automatizada de combinações.

O estudo considera o exercício **Brute Force** do DVWA no nível **Low**, e não confunde esse formulário com o login inicial da aplicação. A sessão autenticada do DVWA, o caminho da instalação, os campos e o comportamento da resposta precisam ser inspecionados antes de qualquer automação. Versões antigas e novas podem diferir.

### Exemplo de lógica, em pseudocódigo

```text
abrir uma sessão autorizada no DVWA do laboratório
confirmar que o exercício Brute Force está no nível Low
para cada uma das três senhas didáticas:
    enviar uma requisição ao formulário com usuário e senha
    preservar os cookies de sessão
    verificar redirecionamento para login, expiração e erros
    classificar a resposta como sucesso, falha ou inconclusiva
    aguardar dois segundos
    interromper em caso de sucesso, bloqueio ou erro inesperado
```

Na implementação Low consultada, o exercício usa parâmetros GET `username`, `password` e `Login`. Uma resposta HTTP 200, isoladamente, não confirma acesso. A validação precisa comparar o conteúdo de sucesso e de falha; uma página de login após expiração de sessão não deve ser classificada como sucesso.

O pseudocódigo não é um programa executável. Ele explicita o fluxo sem supor uma sessão válida, cookies disponíveis ou uma configuração já realizada. Uma futura implementação com Medusa deve consultar a ajuda do módulo web instalado e confirmar suporte a formulário, sessão e critérios de resposta. Um módulo de autenticação HTTP não equivale automaticamente ao envio de um formulário HTML.

**Validação planejada:** comparar uma tentativa sabidamente inválida com uma credencial didática conhecida, depois confirmar o acesso manualmente no próprio exercício. Nenhuma dessas ações foi realizada nesta entrega.

**Prevenção:** limitar tentativas por conta e por origem, aplicar espera progressiva, considerar MFA no sistema de autenticação, usar mensagens genéricas e detectar automação. HTTPS protege o transporte, mas não limita tentativas. Tokens CSRF têm outra finalidade e não substituem controles contra adivinhação de senhas.

## 7. Cenário SMB: enumeração e password spraying

**Hipótese:** a divulgação de contas pode reduzir a incerteza sobre usuários, enquanto uma senha compartilhada pode comprometer mais de uma conta.

Exemplo de consulta ao servidor do laboratório:

```bash
rpcclient -U '' -N 192.168.56.20 -c 'enumdomusers'
```

A consulta depende das permissões do servidor e pode ser negada. Não há garantia de que retorne usuários. O arquivo `wordlists/usuarios-lab.txt` contém contas fictícias; não é saída dessa consulta. Antes de testar, usar somente contas autorizadas e confirmadas no alvo.

Exemplo de **uma rodada** de spraying com uma senha e três usuários didáticos:

```bash
medusa -h 192.168.56.20 -U wordlists/usuarios-lab.txt -p 'SenhaLab123!' -M smbnt -t 1 -O evidencias/smb.txt
```

O exemplo depende da presença e compatibilidade do módulo `smbnt`. Negociação de versões SMB, domínio e políticas do alvo podem exigir ajustes documentados pela ajuda local. Não alterar a segurança do computador pessoal para permitir protocolos antigos.

**Validação planejada:** confirmar um resultado positivo com autenticação manual e consulta de compartilhamentos autorizados. Por exemplo, `smbclient -L //192.168.56.20 -U aluno` solicita a senha de forma interativa. Confirmar que a sessão não foi mapeada para convidado; listar compartilhamentos por si só não comprova autenticação como o usuário solicitado.

**Prevenção:** restringir SMB à rede necessária, desabilitar acesso anônimo e convidado quando dispensáveis, eliminar senhas compartilhadas, desabilitar protocolos obsoletos e correlacionar falhas distribuídas entre contas. Políticas de bloqueio precisam equilibrar proteção e risco de indisponibilidade.

## 8. Situação dos cenários

| Item | Situação nesta entrega | O que seria registrado na prática |
|---|---|---|
| Ambiente virtual | Planejado, não configurado | Rede, versões e endereços reais |
| FTP | Estudado, não executado | Tentativas, respostas e validação manual |
| DVWA | Fluxo documentado, não executado | Sessão, requisições e critérios de classificação |
| SMB | Roteiro documentado, não executado | Contas autorizadas, respostas e identidade autenticada |
| Mitigações | Recomendações propostas | Comparação antes/depois com o mesmo escopo |

Não se afirma que qualquer credencial seja válida ou que um controle tenha sido eficaz. Essas conclusões exigiriam evidências práticas.

## 9. Aprendizados e reflexão

O estudo mostra que a exposição de um serviço e a fragilidade da autenticação são aspectos diferentes. Identificar uma porta aberta é apenas o início da análise. A validação precisa distinguir rejeição de senha, falha de comunicação, sessão expirada e acesso como convidado.

Também evidencia que força bruta e spraying têm padrões distintos: o primeiro pode concentrar muitas tentativas em uma conta; o segundo distribui uma senha entre várias contas. Por isso, a proteção precisa combinar políticas de senha, restrição de acesso, limitação de tentativas e correlação de eventos.

As wordlists deste projeto servem para explicar a metodologia, não para medir a resistência de um sistema real. Como evolução, seria possível montar o laboratório, registrar uma linha de base e repetir os mesmos testes após aplicar controles, respeitando os limites estabelecidos.

## 10. Organização

```text
README.md
docs/registro-de-teste.md
docs/entrega-dio.md
wordlists/usuarios-lab.txt
wordlists/senhas-lab.txt
images/README.md
.gitignore
```

## 11. Referências

- [Kali Linux: Medusa, módulos e opções](https://www.kali.org/tools/medusa/)
- [DVWA: repositório e instruções oficiais](https://github.com/digininja/DVWA)
- [DVWA: código do exercício Brute Force em nível Low](https://github.com/digininja/DVWA/blob/master/vulnerabilities/brute/source/low.php)
- [Nmap: manual oficial](https://nmap.org/book/man.html)
- [VirtualBox: redes virtuais](https://www.virtualbox.org/manual/ch06.html)
- [Samba: rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html)
- [Samba: smbclient](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html)

Fontes consultadas para fundamentar o roteiro; exemplos permanecem sem validação prática. Este projeto não declara conclusão das videoaulas, atividade que deve ser realizada e conferida pelo participante.
