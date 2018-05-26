Problema
--------

Para usar o CUDA no Kaldi, precisamos de uma determinada versão do gcc
instalada em nosso sistema. Como o Manjaro é uma distribuição rolling
release, surgem algumas dificuldades para garantir isso. Esse post
mostra um resumo do que pode ser feito para contornar essa situação.

Solução
-------

1. Instalar o driver da nvidia pela interface gráfica do Hardware 
configuration 

2. Instalar o cuda/cudnn pelo pacman.
```pacman -S cuda cudnn'''

3. Instalar o gcc6 e o gcc6-fortran pelo yaourt (o Cuda versão 9 é incompatível com
gcc superior a 7).
```yaourt gcc6'''

4. No diretório tools do kaldi, executar o make indicando que os compiladores
de C e C++ serão o gcc-6 e o g++-6
```make CXX=g++-6 CC=gcc-6'''

5. Alterar a instrução de compilação do openblas no makefile para 
incluir CC=gcc-6 e FC=gfortran-6. Essa instrução deverá ficar parecida
com:
```$(MAKE) PREFIX=`pwd`/OpenBLAS/install CC=gcc-6 FC=gfortran-6 $(fortran_opt) DEBUG=1 USE_THREAD=1 NUM_THREADS=64 -C OpenBLAS all install'''

6. Compilar OpenBLAS:
```
export LD_LIBRARY_PATH=/usr/lib/gcc/x86_64-pc-linux-gnu/6.4.1/
make openblas
'''

7. No diretório src, executar o configure com os seguintes parâmetros:
```CXX=g++-6 ./configure --openblas-root=../tools/OpenBLAS/install --cudatk-dir=/opt/cuda'''

8. Alterar o arquivo kaldi.mk, incluindo as flags -L e -Wl,-rpath na variável 
OPENBLASLIBS. Essas flags devem indicar onde estão as bibliotecas do g++-6 que serão usadas em tempo de compilação e de execução.
```OPENBLASLIBS = -L/media/dados/git/kaldi-jcsilva/tools/OpenBLAS/install/lib -lopenblas -L/usr/lib/gcc/x86_64-pc-linux-gnu/6.4.1/ -lgfortran -Wl,-rpath=/media/dados/git/kaldi-jcsilva/tools/OpenBLAS/install/lib -Wl,-rpath=/usr/lib/gcc/x86_64-pc-linux-gnu/6.4.1/'''

9. Compilar
```make clean -j; make depend -j; make -j 4'''

