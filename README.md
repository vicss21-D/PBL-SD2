<!DOCTYPE html>
<html lang="pt-br">

<body>

  <h1>Projeto de Coprocessador Gráfico HPS/FPGA (v2.0)</h1>

  <p>Este projeto implementa um sistema embarcado na placa DE1-SoC para redimensionamento de imagens (Zoom In/Out).</p>
  <p>A solução utiliza um design híbrido Hardware/Software, onde o HPS (ARM) executa a lógica de controle principal em C e Assembly, e a FPGA executa um coprocessador gráfico customizado para o processamento de pixels.</p>

  <nav class="toc">
      <h3>Sumário</h3>
      <ul>
          <li>
              <a href="#h2-1">1. Arquitetura e Estratégia do Projeto</a>
              <ul>
                  <li><a href="#h3-1-1">1.1. Definição do Problema e Objetivos</a></li>
                  <li><a href="#h3-1-2">1.2. Requisitos do Projeto</a></li>
                  <li><a href="#h3-1-3">1.3. Estratégia da Solução (O "Porquê")</a></li>
              </ul>
          </li>
          <li>
              <a href="#h2-2">2. Detalhamento dos Módulos</a>
              <ul>
                  <li><a href="#h3-2-1">2.1. Módulo 1: O Coprocessador (Hardware/Verilog)</a></li>
                  <li><a href="#h3-2-2">2.2. Módulo 2: O Driver/API (Software/Assembly)</a></li>
                  <li><a href="#h3-2-3">2.3. Módulo 3: A Aplicação (Software/C)</a></li>
                  <li><a href="#h3-2-4">2.4. Comunicação e Sincronia</a></li>
              </ul>
          </li>
          <li>
              <a href="#h2-3">3. Manuais e Resultados</a>
              <ul>
                  <li><a href="#h3-3-1">3.1. Manual do Usuário (Operação)</a></li>
                  <li><a href="#h3-3-2">3.2. Manual do Sistema (Engenharia)</a></li>
              </ul>
          </li>
      </ul>
  </nav>
  <hr>

  <h2 id="h2-1">1. Arquitetura e Estratégia do Projeto</h2>
  <p>Esta seção aborda a arquitetura geral do sistema e as decisões de design (o "porquê") que levaram à solução implementada.</p>

  <h3 id="h3-1-1">1.1. Definição do Problema e Objetivos</h3>
  <p>O objetivo principal é criar um módulo embarcado para redimensionamento de imagens em escala de cinza (320x240, 8 bits/pixel) em tempo real, simulando interpolação visual. O controle deveria ser feito por uma aplicação de alto nível (em C), que comanda um driver de baixo nível (em Assembly), que por sua vez controla um coprocessador customizado (em Verilog/FPGA).</p>

  <h3 id="h3-1-2">1.2. Requisitos do Projeto</h3>
  <p>O sistema foi dividido em duas etapas principais, culminando em uma aplicação funcional:</p>

  <h4>Etapa 2: A API (Assembly)</h4>
  <ol>
      <li>O código da API deve ser escrito em linguagem Assembly (ARMv7-A).</li>
      <li>Deverão ser implementados os comandos da ISA (Instruction Set Architecture) do coprocessador.</li>
      <li>As imagens (escala de cinza, 8 bits/pixel) devem ser lidas e transferidas para o coprocessador.</li>
      <li>O coprocessador deve ser compatível com o HPS (ARM).</li>
  </ol>

  <h4>Etapa 3: A Aplicação (C)</h4>
  <ol>
      <li>O código da aplicação deve ser escrito em linguagem C.</li>
      <li>O driver (biblioteca Assembly) deve ser linkado ao código C.</li>
      <li>A aplicação deverá ter uma interface de texto para:
          <ul>
              <li>Carregar arquivo bitmap.</li>
              <li>Selecionar algoritmos de zoom.</li>
              <li>Controlar zoom in (+) e zoom out (-).</li>
          </ul>
      </li>
  </ol>

  <h3 id="h3-1-3">1.3. Estratégia da Solução (O "Porquê")</h3>
  <p>A arquitetura foi desenhada para otimizar a divisão de tarefas entre software (flexibilidade) e hardware (velocidade):</p>

  <h4>1.3.1 HPS (Processador ARM):</h4>
  <ul>
      <li>
          <p><strong>Aplicação (C):</strong> A linguagem C foi escolhida como "maestro" por sua alta flexibilidade. Tarefas complexas como parsing de arquivos (.bmp) e gerenciamento de interface de usuário (a 2° parte da implementação, na sequência do atual problema) são triviais em C. Além do mais, com flags de verificação ao longo de todo o código, foi possível aproveitar a flexibilidade da linguagem para a testagem do projeto.</p>
          <p>Segue imagem abaixo demonstrando amostragem de falhas na transcrição dos pixels, programada para ajudar o grupo a otimizar esse processo e enxergar onde, exatamente, o código está apresentando erros.</p>
          <pre><code>ERRO: write_pixel falhou no pixel 5721 (codigo -2)
ERRO: write_pixel falhou no pixel 5722 (codigo -2)
ERRO: write_pixel falhou no pixel 5722 (codigo -2)
ERRO: write_pixel falhou no pixel 5723 (codigo -2)
ERRO: write_pixel falhou no pixel 5723 (codigo -2)
ERRO: write_pixel falhou no pixel 5724 (codigo -2)
ERRO: write_pixel falhou no pixel 5724 (codigo -2)
ERRO: write_pixel falhou no pixel 5725 (codigo -2)
ERRO: write_pixel falhou no pixel 5725 (codigo -2)
ERRO: write_pixel falhou no pixel 5726 (codigo -2)
ERRO: write_pixel falhou no pixel 5726 (codigo -2)
ERRO: write_pixel falhou no pixel 5727 (codigo -2)
ERRO: write_pixel falhou no pixel 5727 (codigo -2)
ERRO: write_pixel falhou no pixel 5728 (codigo -2)
ERRO: write_pixel falhou no pixel 5728 (codigo -2)
ERRO: write_pixel falhou no pixel 5729 (codigo -2)
ERRO: write_pixel falhou no pixel 5729 (codigo -2)
ERRO: write_pixel falhou no pixel 5730 (codigo -2)
ERRO: write_pixel falhou no pixel 5738 (codigo -2)
ERRO: write_pixel falhou no pixel 5732 (codigo -2)
ERRO: write_pixel falhou no pixel 5731 (codigo -2)
</code></pre>
      </li>
      <li>
          <p><strong>Driver (Assembly):</strong> O Assembly (api.s) atua como a camada de driver, ou seja, é a parte que comunica o dispositivo (placa DE1-SoC) com o sistema operacional vigente (Linux Ubuntu). É usado para a interface direta com o hardware, mapeamento da memória virtual (/dev/mem) e, crucialmente, para garantir a sincronia entre o HPS (operando a 800MHz) e a FPGA (operando a 50MHz).</p>
          <p>O clock da placa é dobrado para a instância do HPS e reduzido à sua metade para instância do VGA pelo módulo PLL, já disponibilizado pela Intel.</p>
      </li>
  </ul>

  <h4>1.3.2 FPGA (Hardware):</h4>
  <ul>
      <li>
          <p><strong>Coprocessador (Verilog):</strong> Nas tarefas de processamento não há qualquer forma de paralelismo ou pipeline, mas, apesar da sua simplicidade, atende aos requisitos propostos pelo problema. Vale ressaltar que ao executar essa arquitetura não temos o máximo de otimização, pois há o gargalo da comunicação e da espera síncrona, o código estará sempre esperando determinada instrução finalizar (ativação da flag DONE, em nível alto, após 3 ciclos de clock - 3x100 MHz ou 30 ns) em ordem de seguir para a próxima.</p>
          <p>Nesse prisma, sabendo como esse tipo de fluxo pode culminar na perda de dados ou na dessincronização dos processos, flags, como a citada acima, foram implementadas no corpo de toda a performance do código, forçando os processos esperarem o seu momento de execução. O coprocessador cuida da lógica de zoom, armazenamento de pixels e da exibição VGA.</p>
          <ul>
              <li><strong>Lógica de zoom:</strong> É controlada pelo bloco UEA (Unidade de Execução de Algorítmos) responsável por realizar os cálculos de endereçamento, além de operar os pixels de modo a atender ao algoritmo escolhido.</li>
              <li><strong>Armazenamento dos pixels:</strong> a partir dos cálculos de endereçamento do UEA, o bloco Unidade de Controle gerencia o fluxo de dados entre o HPS e a memória (RAM dual port) com ajuda das flags (Error, Done, Max zoom e Min zoom). Assim, nenhum dado é perdido e a fluência do código é respeitada pois nele não há pipeline e as instruções são sequenciais, els precisam umas das outras para obter os resultados esperados.</li>
              <li><strong>Exibição VGA:</strong> Controlada pelo Módulo VGA, exibe as informações da memória RAM 2 na tela através da porta VGA Presente na DE1-SoC.</li>
          </ul>
      </li>
  </ul>

  <h4>1.3.3 A Ponte (HPS-FPGA):</h4>
  <p>A comunicação é feita pela Lightweight HPS-to-FPGA (Lw_H2F) Bridge (0xFF200000). Esta ponte permite que o HPS (software) veja o coprocessador (hardware) como um simples conjunto de endereços de memória.</p>
  <p><strong>Como isso é feito?</strong></p>
  <ul>
      <li>Utilizando os pios do qsys. Eles funcionam da seguinte forma: são 3 PIOs, dois de entrada- um com o sinal de enable, para começar o processamento do algoritmo; e outro é um de 29 bits, sendo os 3 primeiros o OPcode, os 17 seguintes o endereço na memória RAM 2, o bit 20 foi definido como seletor de memória (não é utilizado) e, por fim, os 8 últimos bits carregam a informação do pixel.</li>
      <li>Por fim, um PIO de saída, que traz os 4 bits das flags, as quais são ativas em nível alto e comandam o andamento do código.</li>
      <li><strong>Onde está no código?</strong> </li>
  </ul>

  <hr>

  <h2 id="h2-2">2. Detalhamento dos Módulos</h2>
  <p>Esta seção detalha cada componente principal, como eles funcionam e como se comunicam.</p>

  <h3 id="h3-2-1">2.1. Módulo 1: O Coprocessador (Hardware/Verilog)</h3>
  <p>O coprocessador é o núcleo de processamento na FPGA (main main_inst). Sua interface com o HPS foi otimizada para apenas três barramentos PIO:</p>
  <ol>
      <li><strong>PIO_INSTR_OFS (Escrita):</strong> Este é o barramento de comando principal.
          <ul>
              <li>Para comandos de algoritmo (ex: NearestNeighbor), ele recebe um opcode simples (ex: 3).</li>
              <li>Para o comando STORE, ele recebe um pacote de dados de 29 bits que combina o opcode, o endereço e o pixel. O Verilog é responsável por decodificar este pacote.</li>
          </ul>
      </li>
      <li><strong>PIO_CONTROL_OFS (Escrita):</strong> Contém o bit ENABLE (pulso para iniciar a operação).</li>
      <li><strong>PIO_FLAGS_OFS (Leitura):</strong> Contém os bits de status (DONE, ERROR, MAX, MIN).</li>
  </ol>

  <h3 id="h3-2-2">2.2. Módulo 2: O Driver/API (Software/Assembly)</h3>
  <p>O arquivo api.s é a biblioteca de driver de baixo nível. Suas funções incluem lógica de robustez e sincronização.</p>
  <ul>
      <li><strong>Inicialização (API_initialize):</strong>
          <ul>
              <li>Mapeia a ponte HPS-FPGA.</li>
              <li>Retorna um código de status negativo em caso de falha (-1 para open, -2 para mmap), em vez de apenas NULL.</li>
          </ul>
      </li>
      <li><strong>Comunicação (ASM_Store):</strong> Esta é η função mais crítica.
          <ul>
              <li><strong>Empacotamento:</strong> Constrói o pacote de 29 bits (Opcode + Endereço + Pixel).</li>
              <li><strong>Barreira de Memória:</strong> Executa uma instrução DMB sy (Data Memory Barrier) antes de pulsar o ENABLE. Isso é vital para forçar o processador HPS a completar a escrita na memória (o STR) antes de prosseguir. Sem isso, o HPS (que faz out-of-order execution) poderia pulsar o ENABLE antes que o pacote de instrução chegasse à FPGA, causando falhas.</li>
              <li><strong>Timeout:</strong> Inicia um loop de polling, ou seja, de verificação contínua, que decrementa um contador (TIMEOUT_LIMIT). Esse contador serve para que o código em C não fique congelado por problemas na FPGA; o algoritmo dá um limite de tempo, o qual, se for excedido, emite um erro de timeout. Se o FLAG_DONE não for recebido a tempo, o loop é interrompido, e a função retorna um código de erro (-2).</li>
          </ul>
      </li>
      <li><strong>Funções Públicas (API):</strong>
          <ul>
              <li><strong>ASM_Store(addr, pixel):</strong> Função Síncrona (Bloqueante). Retorna 0 em sucesso, ou um código de erro negativo.</li>
              <li><strong>Nearest Neighbor(), Pixel Replication(), etc.:</strong> Funções Assíncronas. Apenas definem o opcode da instrução no PIO_INSTR_OFS.</li>
              <li><strong>ASM_Get_Flag_Done(), etc.:</strong> Expõem o leitor de flags para o C.</li>
          </ul>
      </li>
  </ul>

  <h3 id="h3-2-3">2.3. Módulo 3: A Aplicação (Software/C)</h3>
  <p>O arquivo main.c é o "maestro" de alto nível.</p>
  <ul>
      <li><strong>Carregamento:</strong> É responsável por abrir (fopen) o arquivo image.bmp, pular o cabeçalho (fseek de 1078 bytes) e carregar os 76800 pixels (320x240) para um buffer na RAM 1 (fread).</li>
      <li><strong>Lógica de Controle:</strong> O C implementa a lógica assíncrona para executar um algoritmo:
          <ol>
              <li>Chama a API (ex: NearestNeighbor()) para definir a instrução (não-bloqueante).</li>
              <li>Chama ASM_Pulse_Enable() para iniciar o hardware (não-bloqueante).</li>
              <li>Entra em um loop while (ASM_Get_Flag_Done() == 0) para fazer polling (sondagem) do FLAG_DONE, e consequentemente, dos retornos do código.</li>
              <li>Quando a flag retorna 1, o C sabe que o hardware terminou.</li>
          </ol>
      </li>
  </ul>

  <h3 id="h3-2-4">2.4. Comunicação e Sincronia</h3>
  <p>A sincronização de dados (para STORE) é síncrona (o Assembly espera pelo FLAG_DONE e tem timeout). A sincronização de comandos (para algoritmos) é assíncrona (o C faz o polling).</p>
  <p>A maior complexidade resolvida foi a transferência de dados. Ao invés de três escritas separadas (Instrução, Endereço, Dado), que poderiam ser reordenadas pelo HPS ou criar corridas na FPGA de 50MHz, a solução agora é:</p>
  <ol>
      <li>HPS (Assembly): Monta um "pacote" de 29 bits.</li>
      <li>HPS (Assembly): Escreve o pacote no PIO_INSTR_OFS.</li>
      <li>HPS (Assembly): Executa DMB (Garante que a escrita terminou).</li>
      <li>HPS (Assembly): Pulsa ENABLE.</li>
      <li>FPGA (Verilog): Vê o pulso, lê o pacote de 29 bits, decodifica (Opcode=STORE, Endereço=X, Dado=Y) e executa a escrita.</li>
      <li>FPGA (Verilog): Levanta FLAG_DONE.</li>
      <li>HPS (Assembly): Vê o FLAG_DONE e retorna 0 (Sucesso).</li>
  </ol>

  <hr>

  <h2 id="h2-3">3. Manuais e Resultados</h2>
  <p>Esta seção contém as instruções de uso (para um usuário) e as especificações técnicas (para um engenheiro).</p>

  <h3 id="h3-3-1">3.1. Manual do Usuário (Operação)</h3>
  <p>Este guia permite que qualquer usuário execute a solução.</p>

  <h4>3.1.1. Pré-requisitos</h4>
  <ul>
      <li>Placa DE1-SoC com a distribuição Linux (Ubuntu 24.04.2 LTS) inicializada.</li>
      <li>Monitor VGA (ex: Philips 191 EL) e Teclado USB (ex: Lenovo KU-1619) conectados.</li>
      <li>Conexão de rede (SSH/SCP) com a placa.</li>
  </ul>

  <h4>3.1.2. Processo de Execução</h4>
  <p><em>{trazer fotos e prints do terminal, detalhar mais o ligamento e o funcionamento da placa}</em></p>
  <p>Siga estes passos para compilar e executar o projeto:</p>

  <h5>1. Programar a FPGA:</h5>
  <ul>
      <li>Insira os cabos de rede e de alimentação da placa (ao todo serão 3);</li>
      <li>Ligue a placa e aguarde ela finalizar o processo de ligamento;</li>
      <li>Compile o projeto no Intel Quartus Prime (v23.1std.0. Link de download) para gerar o arquivo .rbf (ex: ghrd.rbf);</li>
      <li>Descarregue o código na placa, através do menu programer;</li>
  </ul>

  <p>Copiar o código no endereço da placa por meio do navegador de arquivos;</p>
  <p>Selecione a pasta que comporta o seu projeto, e copie os códigos em assembly, em C e o makefile do verilog</p>
  <p>Abra seu terminal e siga o passo a passo descrito nas imagens</p>

  <pre><code>aluno@LEDS-54163: $ ssh aluno@172.65.213.123
aluno@172.65.213.123's password: </code></pre>
  <p>Com o comando ssh acesse a placa com o nome de usuário informado previamente e insira a senha.</p>
  
  <pre><code>aluno@LEDS-54163:$ ssh aluno@172.65.213.123
aluno@172.65.213.123's password:
Last login: Thu Jan 1 01:56:17 1970 from leds-54163.lan
aluno@de1soc123:~$ cd TEC499/TP04/G02/ASM/</code></pre>
  <p>Apesar de a senha não aparecer, ela está sendo registrada, não se preocupe! O cmd retornará o endereço do último acesso, edite se necessário. Não esqueça de usar o comando 'cd'.</p>

  <pre><code>aluno@de1soc123:~/TEC499/TP04/G02/ASM$ ls
api.h coprocessador_2_lib.s exe lib.o lib.s main.c makefile
aluno@de1soc123:~/TEC499/TP04/G02/ASM$ </code></pre>
  <p>Digite o comando ls para acessar os últimos arquivos copiados (opcional)</p>

  <pre><code>aluno@de1soc123:~/TEC499/TP04/G02/ASM$ ls
api.h coprocessador_2_lib.s exe lib.o lib.s main.c makefile
aluno@de1soc123:~/TEC499/TP04/G02/ASM$ sudo su
[sudo] password for aluno: </code></pre>
  <p>Com o endereço da pasta do projeto, você deve executar os arquivos como super usuário, para isso, digite o comando "sudo su". Depois, insira η senha.</p>

  <pre><code>aluno@de1soc123:~/TEC499/TP04/G02/ASM$ ls
api.h coprocessador_2_lib.s exe lib.o lib.s main.c makefile
aluno@de1soc123:~/TEC499/TP04/G02/ASM$ sudo su
[sudo] password for aluno:
root@de1soc123:/home/aluno/TEC499/TP04/G02/ASM# make run</code></pre>
  <p>Agora, digite o comando "make run" para finalmente executar o código e acessar a interface interativa em C.</p>

  <pre><code>Montando lib.s
lib.s: Assembler messages:
lib.s: Warning: end of file not at end of a line; newline inserted
--- Compilando e Ligando (C) main.c
main.c: In function 'enviar_imagem_para_fpga :
main.c:180:5: warning: implicit declaration of function 'usleep' [-Wimplicit-function-declaration]
Executando</code></pre>
  <p>Após os passos acima, você deve visualizar essa mensagem: alegre-se! O código está sendo compilado e retornará a função principal.</p>

  <h4>3.1.3. Como usar o algoritmo passo a passo</h4>
  
  <p><strong>Passo 1:</strong> escolha a opção 1, para inicializar a API. esteja atento(a) às mensagens do estado dela, do buffer e do vga (elas ficam logo abaixo de "MENU DE TESTE DA API". Você deve ver essa mensagem:</p>
  <pre><code>=== MENU DE TESTE DA API ===
ESTADO: API [OFF] | Buffer C [VAZIA] | FPGA VRAM [VAZIA]
1. Inicializar API
Carga de Imagem (Buffer C)
2. Carregar Imagem BMP
3. Gerar Gradiente
4. Enviar Imagem (Buffer C -> FPGA VRAM)
Comandos/Algoritmos (FPGA)
5. NearestNeighbor
6. PixelReplication
7. Decimation
8. BlockAveraging
9. Atualizar (ASM_Refresh)
10. RESET
0. Encerrar API e Sair
Escolha uma opcao: 1

=== PASSO 1: Inicializando API (API_initialize)
>>> SUCESSO: API inicializada.
Pressione Enter para continuar...</code></pre>

  <p><strong>Passo 2:</strong> Carregue a imagem (opção 2) colocando o nome do arquivo dela. Esteja atento(a) para a escrita, pois ela deve ser exatamente igual à que você colocou ao criar o bmp na pasta do usuário. A opção alternativa é gerar o gradiente (opção 3)</p>
  <pre><code>Escolha uma opcao: 2
=== PASSO 2: Carregando Imagem BMP ===
Digite o nome do arquivo BMP: Xadrez.bmp
Carregando.... OK!
>>> SUCESSO: Imagem BMP carregada no buffer C.
Pressione Enter para continuar...</code></pre>

  <p><strong>Passo 3:</strong> Envie a imagem, ou o gradiente, ao Buffer (opção 4). Após a mensagem de sucesso, o usuário pode usar os algoritmos de zoom in/out livremente.</p>
  <pre><code>=== MENU DE TESTE DA API ===
ESTADO: API [ON] | Buffer C [CARREGADA] | FPGA VRAM [VAZIA]
1. Inicializar API
Carga de Imagem (Buffer C)
2. Carregar Imagem BMP
3. Gerar Gradiente
4. Enviar Imagem (Buffer C -> FPGA VRAM)
Comandos/Algoritmos (FPGA)
5. Nearest Neighbor
6. PixelReplication
7. Decimation
8. BlockAveraging
9. Atualizar (ASM_Refresh)
10. RESET
0. Encerrar API e Sair
Escolha uma opcao: 4

=== PASSO 3: Enviando Imagem para FPGA ===
[C] Enviando 76800 pixels para o FPGA (testando ASM_Store)...
[C] Envio de pixels OK.
[C] Testando ASM_Refresh()...
>>> SUCESSO: Imagem enviada para a VRAM do FPGA.
Pressione Enter para continuar...</code></pre>


  <h4>3.1.4. Resultados Esperados:</h4>
  <ul>
      <li>O terminal de texto exibirá as mensagens de status (mapeamento, leitura, envio, polling, concluído).</li>
      <li>O monitor VGA exibirá a imagem image.bmp (ou o gradiente gerado) processada pelo algoritmo (ex: NearestNeighbor).</li>
      <li>Em caso de zoom máximo/mínimo, a flag usada para sinalizar o limite do processamento será mostrada, e o usuário será encaminhado para duas opções: executar o algoritmo contrário ao da condição extrapolada ou resetar a imagem.</li>
  </ul>
  <p>Observe:</p>
  <pre><code>Escolha uma opcao: 7
ERRO: A 'Flag de Zoom Minimo (Min_Zoom) esta ATIVA.
Nao e possivel executar mais algoritmos de Zoom OUT.
Tente um Zoom IN' (5, 6) ou 'Reset' (10).
Pressione Enter para continuar...</code></pre>
  <pre><code>Escolha uma opcao: 5
ERRO: A 'Flag de Uso Maximo' (Max_Zoom) esta ATIVA.
Nao e possivel executar mais algoritmos de Zoom IN.
Tente um 'Zoom OUT' (7, 8) ou 'Reset' (10).
Pressione Enter para continuar...</code></pre>

  <hr>

  <h3 id="h3-3-2">3.2. Manual do Sistema (Engenharia)</h3>
  <p>Este manual contém as especificações técnicas para manutenção, versionamento ou debug do projeto.</p>

  <h4>3.2.1. Especificações do Ambiente</h4>
  <ul>
      <li><strong>Hardware (Placa):</strong> Terasic DE1-SoC (Cyclone V SE 5CSEMA5F31C6N)</li>
      <li><strong>Hardware (Testes):</strong>
          <ul>
              <li>Monitor: Philips 191 EL (640x480)</li>
              <li>Teclado: Lenovo KU-1619 (5V)</li>
          </ul>
      </li>
      <li><strong>Software (FPGA):</strong> Intel Quartus Prime v23.1std.0</li>
      <li><strong>Software (HPS OS):</strong> Ubuntu 24.04.2 LTS</li>
      <li><strong>Software (Toolchain):</strong> GCC 13.3.0 (arm-linux-gnueabihf)</li>
      <li><strong>Parâmetros Imagem:</strong> 320x240 pixels (76800 bytes)</li>
  </ul>

  <h4>3.2.2. A Relação Assembly-Coprocessador (Interface HPS/FPGA)</h4>
  <p>A comunicação HPS-FPGA ocorre no endereço base 0xFF200000 (Ponte LW). A nova arquitetura de PIO foi otimizada para minimizar corridas de barramento:</p>
  
  <table>
      <thead>
          <tr>
              <th>Offset (Hex)</th>
              <th>PIO (Hardware)</th>
              <th>Propósito (Software)</th>
              <th>R/W</th>
          </tr>
      </thead>
      <tbody>
          <tr>
              <td>0x00</td>
              <td>pio_instruction</td>
              <td>Barramento de Instrução/Dados (29 bits).</td>
              <td>Write</td>
          </tr>
          <tr>
              <td>0x10</td>
              <td>pio_control</td>
              <td>Bit 0: ENABLE. Bit 1: SEL_MEM.</td>
              <td>Write</td>
          </tr>
          <tr>
              <td>0x20</td>
              <td>pio_flags</td>
              <td>Bits de status (DONE, ERROR, MAX, MIN).</td>
              <td>Read</td>
          </tr>
      </tbody>
  </table>

  <h5>Detalhamento Funcional: A Interface de Comando (PIO_INSTR_OFS)</h5>
  <p>O ponto nevrálgico da comunicação HPS-FPGA é o barramento PIO_INSTR_OFS (localizado em [BASE] + 0x00). Este barramento foi projetado para operar em dois modos distintos para otimizar o fluxo de dados e controle entre o HPS (950MHz) e a FPGA (50MHz).</p>
  <p>O modo de operação é determinado pelo tipo de comando que o software (C/Assembly) envia.</p>

  <h6>A. Modo 1: Comandos de Algoritmo (Fluxo Assíncrono)</h6>
  <p>Este modo é usado para instruções que configuram o estado do coprocessador, como Nearest Neighbor, PixelReplication, Decimation, e ASM_Reset.</p>
  <ul>
      <li><strong>Bloco Funcional:</strong> _ASM_Set_Instruction (chamada pelas funções públicas da API).</li>
      <li><strong>Dados de Entrada (Assembly):</strong> Um opcode de 3 bits (ex: INSTR_NHI_ALG = 3).</li>
      <li><strong>Processamento (Assembly):</strong> A função é "burra" (simples). Ela apenas move o opcode para um registrador e o escreve (STR) diretamente no barramento PIO_INSTR_OFS.</li>
      <li><strong>Dados de Saída (para a FPGA):</strong> Um valor de 32 bits onde apenas os 3 bits menos significativos são relevantes (ex: 0x00000003).</li>
  </ul>
  <p><strong>Fluxo de Dados e Controle:</strong></p>
  <ol>
      <li><strong>Dado:</strong> O main.c chama NearestNeighbor(). A api.s escreve 3 no PIO_INSTR_OFS.</li>
      <li><strong>Controle (Separado):</strong> A função Assembly retorna imediatamente ao main.c (é não-bloqueante).</li>
      <li>O main.c deve, em uma chamada separada, invocar ASM_Pulse_Enable().</li>
      <li>A FPGA, ao detectar o pulso ENABLE, lê o barramento PIO_INSTR_OFS, vê o 3, e sua Máquina de Estados Finitos (FSM) interna transiciona para o estado "Executar Nearest Neighbor".</li>
  </ol>
  <p>Este fluxo é assíncrono porque o envio do dado (o opcode) e o sinal de controle (o ENABLE) são duas ações de software distintas, gerenciadas pelo main.c.</p>

  <h6>B. Modo 2: Comando de Escrita de Dados (Fluxo Síncrono)</h6>
  <p>Este modo é usado exclusivamente pela função ASM_Store para transferir os 76.800 pixels para a memória interna (FrameBuffer) da FPGA. Ele é otimizado para resolver a disparidade de clock HPS-FPGA e evitar condições de corrida.</p>
  <ul>
      <li><strong>Bloco Funcional:</strong> ASM_Store.</li>
      <li><strong>Dados de Entrada (Assembly):</strong> R0 (endereço do pixel) e R1 (dado do pixel).</li>
      <li><strong>Processamento (Assembly):</strong> A função executa um "empacotamento de bits" (bit-packing) para fundir três informações (Opcode, Endereço, Dado) em um único pacote de 29 bits.</li>
  </ul>
  <p>O código-chave para este processamento, extraído do api.s, é:</p>
  <p>O Assembly constrói um pacote de 29 bits e o escreve no [BASE]+0x00. O Verilog decodifica este pacote, que é estruturado da seguinte forma (baseado no ASM_Store):</p>
  <ul>
      <li><strong>Bits [2:0]:</strong> Opcode (ex: INSTR_STORE = 2)</li>
      <li><strong>Bits [19:3]:</strong> Endereço da Memória (17 bits, r0 &lt;&lt; 3)</li>
      <li><strong>Bits [28:20]:</strong> Dado do Pixel (8 bits, r1 &lt;&lt; 20)</li>
  </ul>
  <p>O main.c chama ASM_Store(endereco, pixel), e o Assembly executa:</p>
  <pre><code>@ Fragmento do código
ASM_PACKET_CONSTRUCTION:
  @ R0=endereco, R1=pixel_data
  MOV r2, #STORE_OPCODE  @ r2 = [00...0010]
  
  @ Endereço
  LSL r3, r0, #3         @ r3 = [endereco &lt;&lt; 3]
  ORR r2, r2, r3         @ r2 = [00...0][endereco][010]
  
  @ Pixel
  LSL r3, r1, #20        @ r3 = [pixel &lt;&lt; 20]
  ORR r2, r2, r3         @ r2 = [0][pixel][0][endereco][010]
  
  STR r2, [r4, #PIO_INSTR_OFS] @ Envia o pacote de 29 bits</code></pre>
  <p><strong>Dados de Saída (para a FPGA):</strong> Um único valor de 32 bits (ex: 0b...[pixel_data]...[mem_address].[010]) que representa o pacote de instrução completo.</p>
  <p><strong>Fluxo de Dados e Controle:</strong></p>
  <ol>
      <li><strong>Dado:</strong> O pacote de 29 bits é escrito (STR) no PIO_INSTR_OFS.</li>
      <li><strong>Controle (Sincronização):</strong> A instrução DMB sy (Data Memory Barrier) é executada. Ela força o pipeline do HPS a garantir que a escrita (STR) anterior esteja concluída e visível na ponte antes de qualquer instrução subsequente.</li>
      <li><strong>Controle (Ativação):</strong> O _pulse_enable_safe é chamado internamente.</li>
      <li>A FPGA, ao ver o ENABLE, lê o barramento PIO_INSTR_OFS, "trava" (latches) o pacote de 29 bits, e uma FSM de decodificação no Verilog "desempacota" os bits ([2:0], [19:3], [27:20]) para executar a escrita na memória interna.</li>
      <li><strong>Controle (Confirmação):</strong> A ASM_Store não retorna; ela entra em polling (com timeout) e espera ativamente o FLAG_DONE da FPGA antes de retornar o controle ao main.c.</li>
  </ol>

  <h4>3.2.3. API e Sincronização</h4>
  
  <table>
      <thead>
          <tr>
              <th>Função (Protótipo C)</th>
              <th>Descrição (Assembly)</th>
              <th>Tipo</th>
              <th>Retorno (Sucesso)</th>
              <th>Retorno (Erro)</th>
          </tr>
      </thead>
      <tbody>
          <tr>
              <td><code>int API_initialize(void);</code></td>
              <td>Abre e mapeia /dev/mem (mmap).</td>
              <td>Inicialização</td>
              <td>0 (&gt; 0 Ponteiro)</td>
              <td>-1 (Open) ou -2 (Mmap)</td>
          </tr>
          <tr>
              <td><code>void API_close(void);</code></td>
              <td>Fecha e desmapeia /dev/mem (munmap).</td>
              <td>Limpeza</td>
              <td>void</td>
              <td>N/A</td>
          </tr>
          <tr>
              <td><code>int ASM_Store(unsigned int a, unsigned char d);</code></td>
              <td>Armazena 1 pixel (Síncrono).</td>
              <td>E/S de Dados</td>
              <td>0</td>
              <td>-1 (Endereço), -2 (Timeout), -3 (Erro HW)</td>
          </tr>
          <tr>
              <td><code>void NearestNeighbor(void);</code></td>
              <td>Define o opcode INSTR_NHI_ALG (3).</td>
              <td>Controle (Assíncrono)</td>
              <td>void</td>
              <td>N/A</td>
          </tr>
          <tr>
              <td><code>void PixelReplication(void);</code></td>
              <td>Define o opcode INSTR_PR_ALG (4).</td>
              <td>Controle (Assíncrono)</td>
              <td>void</td>
              <td>N/A</td>
          </tr>
          <tr>
              <td><code>void Decimation(void);</code></td>
              <td>Define o opcode INSTR_NH_ALG (6).</td>
              <td>Controle (Assíncrono)</td>
              <td>void</td>
              <td>N/A</td>
          </tr>
          <tr>
              <td><code>void BlockAveraging(void);</code></td>
              <td>Define o opcode INSTR_BA_ALG (5).</td>
              <td>Controle (Assíncrono)</td>
              <td>void</td>
              <td>N/A</td>
          </tr>
          <tr>
              <td><code>void ASM_Reset(void);</code></td>
              <td>Define o opcode INSTR_RESET (7).</td>
              <td>Controle (Assíncrono)</td>
              <td>void</td>
              <td>N/A</td>
          </tr>
          <tr>
              <td><code>void ASM_Pulse_Enable(void);</code></td>
              <td>Pulsa o pino ENABLE (inicia hardware).</td>
              <td>Controle (Assíncrono)</td>
              <td>void</td>
              <td>N/A</td>
          </tr>
          <tr>
              <td><code>int ASM_Get_Flag_Done(void);</code></td>
              <td>Retorna 1 se FLAG_DONE_MASK (1) está alta.</td>
              <td>Status (Polling)</td>
              <td>1 (Ativo)</td>
              <td>0 (Inativo)</td>
          </tr>
          <tr>
              <td><code>int ASM_Get_Flag_Error(void);</code></td>
              <td>Retorna 1 se FLAG_ERROR_MASK (2) está alta.</td>
              <td>Status (Polling)</td>
              <td>1 (Ativo)</td>
              <td>0 (Inativo)</td>
          </tr>
          <tr>
              <td><code>int ASM_Get_Flag_Max_Zoom(void);</code></td>
              <td>Retorna 1 se FLAG_MAX_ZOOM_MASK (4) está alta.</td>
              <td>Status (Polling)</td>
              <td>1 (Ativo)</td>
              <td>0 (Inativo)</td>
          </tr>
          <tr>
              <td><code>int ASM_Get_Flag_Min_Zoom(void);</code></td>
              <td>Retorna 1 se FLAG_MIN_ZOOM_MASK (8) está alta.</td>
              <td>Status (Polling)</td>
              <td>1 (Ativo)</td>
              <td>0 (Inativo)</td>
          </tr>
      </tbody>
  </table>

  <h4>3.2.4. Sincronização (DMB)</h4>
  <p>A ASM_Store usa DMB sy (Data Memory Barrier) para resolver a diferença de velocidade HPS/FPGA e a execução out-of-order do HPS. O DMB força o processador a "drenar" sua sequência de escrita, garantindo que o STR do pacote de instrução seja concluído na ponte antes que o STR do pulso ENABLE (dentro de _pulse_enable_safe) seja emitido.</p>

  <p><strong>O Fluxo Correto (Com DMB):</strong></p>
  <ol>
      <li><strong>Instrução 1 (STR pacote):</strong> O HPS joga o pacote de 29 bits no buffer de escrita.</li>
      <li><strong>Instrução 2 (DMB sy):</strong> O HPS encontra a barreira e TRAVA (stall). Ele se recusa a pular para a próxima instrução. Ele é forçado a parar e esperar até que o buffer de escrita esteja completamente vazio e ele tenha a confirmação de que o pacote de 29 bits foi escrito e é visível na ponte.</li>
      <li>Somente após a garantia do passo 2, o HPS finalmente "pula a cerca".</li>
      <li><strong>Instrução 3 (BL _pulse_enable_safe):</strong> Agora, de forma segura, o HPS envia o pulso ENABLE.</li>
  </ol>
  <p><strong>Garantia:</strong> Quando a FPGA vê o ENABLE, o DMB nos deu a garantia absoluta de que o PACOTE de 29 bits já está lá, estável e pronto para ser lido. Você eliminou a condição de corrida.</p>

  <h4>3.2.5. Análise dos Resultados e Validação</h4>
  <p>A validação do projeto foi realizada executando o procedimento descrito no Manual do Usuário (3.1).</p>
  <p>O console de texto (via SSH) exibiu a sequência correta de operações, confirmando que a aplicação main.c executou, carregou a imagem e que as funções da api.s (como ASM_Store) retornaram sucesso (0).</p>
  <p><strong>Validação do hardware:</strong></p>
  ![animacao_monitor_lento](https://github.com/user-attachments/assets/aa127bd7-3526-46a7-955e-e9caf044b28d)

  </body>
</html>
