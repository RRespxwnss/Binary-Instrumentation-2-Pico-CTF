# Binary-Instrumentation-2 Pico-CTF
I've been learning more Windows API functions to do my bidding. Hmm... I swear this program was supposed to create a file and write the flag directly to the file. Can you try and intercept the file writing function to see what went wrong?


Abrindo o binário no Ghidra
A primeira coisa que faço em qualquer binário é olhar as seções do PE (Portable Executable). Após importar o arquivo e rodar a análise automática, fui ao Program Tree no canto esquerdo para ver a estrutura do arquivo.
<img width="800" alt="Visao geral do binario no Ghidra" src="https://github.com/user-attachments/assets/a7ae5f30-503f-4e26-98be-d1ad6a1a4200" />
A maioria das seções era completamente normal: .text para código, .rdata para dados somente leitura, .data, .pdata, .rsrc, .reloc. Mas uma chamou atenção imediata: .ATOM. Esse nome não existe no padrão PE — seções com nomes fora do convencional quase sempre escondem conteúdo empacotado ou comprimido.

Investigando a seção .ATOM
Naveguei até o header da seção .ATOM no Listing. Os campos mais importantes revelaram tudo que precisava saber sobre onde os dados estavam:
<img width="784" alt="Header da secao .ATOM mostrando VirtualAddress SizeOfRawData e PointerToRawData" src="https://github.com/user-attachments/assets/6e0cd689-552a-483e-8acc-d2faa4a7f930" />
O campo PointerToRawData = 0x6000 indica o offset no arquivo onde os dados reais da seção começam, e SizeOfRawData = 0x1200 diz que são 4608 bytes. Pressionei G e naveguei para o endereço 14000b000 para ver esses bytes diretamente.

Reconhecendo o formato LZMA
Ao chegar nos dados brutos da seção, os primeiros bytes já entregaram o formato:
<img width="800" alt="Bytes iniciais da secao .ATOM mostrando 5D 00 00 10 - header LZMA" src="https://github.com/user-attachments/assets/7f96e4dd-f556-423b-8d45-5bb94b1acf9e" />
O byte 5D no início é o properties byte do LZMA, seguido do tamanho do dicionário e do tamanho descomprimido. Diferente de GZIP (1F 8B) ou ZIP (50 4B), o LZMA não tem magic bytes fixos — o reconhecimento vem da combinação do byte 0x5D com a estrutura dos bytes seguintes e o contexto de uma seção PE anômala.
5D           → Properties byte (configuração padrão do LZMA)
00 00 10 00  → Dictionary size: 1MB
2C 00 00 00 00 00 00 00  → Tamanho descomprimido: 11.264 bytes
Confirmado: os dados da seção .ATOM são um stream LZMA comprimido.

Extraindo os bytes com script no Ghidra
Para salvar os bytes da seção em disco, usei o Script Manager do Ghidra (Window → Script Manager → botão + → New Python Script):
<img width="800" alt="Script Manager do Ghidra com o novo script Python criado" src="https://github.com/user-attachments/assets/cbd3b21f-a9f2-460f-b4ad-11ace110e8c5" />
<img width="800" alt="Editor do script com o codigo de extracao dos bytes" src="https://github.com/user-attachments/assets/c630c0f9-44f9-4697-9973-48b92a48ae54" />

A conversão b & 0xFF é necessária porque o Ghidra retorna bytes como inteiros Java com sinal — sem isso, bytes maiores que 127 virariam negativos e corrompem o arquivo. Após rodar com F5, o atom.bin apareceu no Desktop.
<img width="800" alt="Console do Ghidra mostrando Salvo apos rodar o script" src="https://github.com/user-attachments/assets/9086ca54-e7a4-4d10-b0de-c7d13a44d52d" />

Descomprimindo no CyberChef
Com o atom.bin em mãos, arrastei o arquivo direto para o CyberChef e montei a receita From Hex → LZMA Decompress:
<img width="1096" alt="CyberChef com a receita From Hex e LZMA Decompress configurada" src="https://github.com/user-attachments/assets/a580b25a-e44e-4126-b742-da9c7187c7c7" />
<img width="1096" alt="CyberChef processando os dados da secao .ATOM" src="https://github.com/user-attachments/assets/d1b4a13b-ebf9-4dda-ab08-52d64824eb0a" />
O output foi um arquivo binário completo. Já no conteúdo descomprimido dá para ver a assinatura MZ e o XML de manifest — estrutura característica de um executável Windows. Havia um segundo .exe inteiro escondido dentro do primeiro.
<img width="1095" alt="Output do CyberChef revelando um segundo executavel PE dentro do primeiro" src="https://github.com/user-attachments/assets/9d372837-dba4-469c-a4c3-abe1a5f156b2" />
Baixei o output clicando no ícone de download no CyberChef e renomeei para inner.exe.

Analisando o executável interno
Com o inner.exe em mãos, abri um novo projeto no Ghidra e importei o arquivo. A análise automática identificou corretamente como PE64 Windows.
<img width="784" alt="Ghidra importando o inner.exe extraido" src="https://github.com/user-attachments/assets/c905f89b-57bc-4937-95ee-bbc9235f3d3b" />
<img width="784" alt="Analise automatica do inner.exe em progresso" src="https://github.com/user-attachments/assets/c0e1b434-0dcd-464c-bd7b-dce4225ebc98" />
<img width="785" alt="inner.exe carregado no Ghidra com as secoes visiveis" src="https://github.com/user-attachments/assets/da6d6154-07fc-406f-bd50-3570a7dc03a7" />

Encontrando a flag em Defined Strings
Fui direto em Window → Defined Strings — esse é sempre o primeiro lugar a procurar em CTFs. O que apareceu confirmou tudo:
<img width="800" alt="Defined Strings do inner.exe mostrando a string Insert path here e a string Base64" src="https://github.com/user-attachments/assets/2ea1cb3e-1fe5-467f-9305-58a67ec63e75" />
Duas strings se destacaram imediatamente:
<Insert path here> — um placeholder que nunca foi substituído pelo caminho real do arquivo de saída. Esse é o bug que impede a flag de ser escrita no disco.
cGljb0NURntmcjFkYV9mMHJfYjFuX2luNXRydW0zbnQ0dGlvbiFfYjIxYWVmMzl9 — uma string em Base64. Em CTFs, strings Base64 em binários quase sempre valem ser decodificadas.

Entendendo o bug no decompilador
Cliquei duas vezes em <Insert path here> para ir até onde essa string é usada no código. No decompilador, o problema ficou evidente:
<img width="800" alt="Decompilador mostrando CreateFileA recebendo Insert path here como caminho" src="https://github.com/user-attachments/assets/881c5992-ee52-4d46-ba01-0ec93f89768c" />
<img width="800" alt="Fluxo completo mostrando CreateFileA falhando e WriteFile nao executando" src="https://github.com/user-attachments/assets/1eaa8390-a21b-48cc-acf9-ae72e05406fb" />
```c
HANDLE hFile = CreateFileA("<Insert path here>", GENERIC_WRITE, ...);
WriteFile(hFile, buffer, size, &bytesWritten, NULL);
```
O caminho passado para CreateFileA é literalmente o texto <Insert path here>. O desenvolvedor deixou o placeholder no código sem substituir pelo caminho real. CreateFileA falha, retorna INVALID_HANDLE_VALUE, e WriteFile não consegue escrever nada — a flag nunca chega ao disco.

Verificando os imports
Para confirmar quais funções da Windows API estavam sendo usadas, olhei os imports do inner.exe:
<img width="800" alt="Imports do inner.exe mostrando WriteFile e CreateFileA do KERNEL32.dll" src="https://github.com/user-attachments/assets/fce9e108-620d-43ea-a072-32de74377424" />
<img width="800" alt="Referencias a WriteFile no codigo do inner.exe" src="https://github.com/user-attachments/assets/94a5e7d6-480b-42a4-863f-621655a0e000" />
WriteFile e CreateFileA do KERNEL32.dll — exatamente as funções mencionadas na descrição do desafio. Isso confirma que o binário interno é o responsável pela escrita do arquivo, e o bug está no caminho passado para CreateFileA.

Decodificando a flag
Com a string Base64 identificada, a decodificação é simples:
<img width="800" alt="Decodificacao da string Base64 revelando a flag picoCTF" src="https://github.com/user-attachments/assets/a6a0df63-58c0-42e7-942d-5c10c2081b67" />

Como funcionou
O bininst2.exe era na verdade um loader seu único propósito era carregar o executável real que estava comprimido com LZMA dentro da seção .ATOM. O executável interno (inner.exe) continha a lógica de criar um arquivo e escrever a flag, mas o caminho do arquivo nunca foi preenchido corretamente pelo desenvolvedor. A flag estava lá o tempo todo, embutida em Base64 no código, esperando para ser encontrada.

picoCTF{fr1da_f0r_b1n_in5trum3nt4tion!_b21aef39}

