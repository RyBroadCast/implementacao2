# Implementação 2

A ideia da implementação era criar um simulador de um manager de memória virtual, como na atividade do livro Operating System Concepts, Silberschatz, A. et al, 10a edição. A atividade consiste em implementar algumas soluções de desempenho para buscar páginas na memória, como uma TLB e uma Page Table. OBS: o LRU não foi implementado, portanto só irá funcionar caso o comando seja "./vm address.txt fifo fifo".

### Como executar:
Através dos comandos:
 - make
 - make run
 - make clean
Através dos comandos:
 - ./vm addresses.txt fifo fifo
 - ./vm addresses.txt fifo lru
 - ./vm addresses.txt lru fifo
 - ./vm addresses.txt lru lru

### Código:
Incluindo bibliotecas e declarando funções e variáveis globais:
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
#include <pthread.h>
#define TLB_size 16

void atualizaTLB();
void checkTLB(int i);

int global_index = 0, page_num = 0, offset = 0, frame = 0, i_tlb = 0, is_in_tlb = 0, tlb_hit;
int TLB[TLB_size][2];


Função int main() com argc e argv para ler as linhas de comando. Declara variáveis e abre os arquivos
c
int main(int argc, char *argv[]) {

  void atualizaTLB();
  int i = 0;
  int archive;
  int value = 0, error = 0;
  int page_table[256], physical_mem[128][256], physical_ad;
  float page_fault = 0, pf_rate = 0, th_rate = 0;
  int is_in_pagetable, translated_files = 0;
  memset(page_table, -1, sizeof(page_table)); // preenche a pagetable com -1
  
  signed char bin_buffer[256];

  FILE *fp, *fbin, *fc;

  fp = fopen(argv[1], "r");
  fc = fopen("correct.txt", "w");

  if (!fp) {
      fprintf(fp, "File not found!");
      fclose(fp);
  }
```

O else, le os arquivos, traduz o page_num e o offset, usa as threads para buscar na TLB:
```c
else {
    while ((fscanf(fp, "%d", &archive)) != EOF){
      is_in_tlb = 0;
      is_in_pagetable = 0;

      //traduz pra binário, separa em pagenum e offset
      page_num = archive >> 8;
      offset =  archive & 255;

      // TLB
      pthread_t threads[16];
      for (int i = 0; i < TLB_size; i++){
        if(pthread_create(&(threads[i]), NULL, checkTLB, i)){
			    printf("A thread %d não foi criada", i);
		    }
      }

      for (int i = 0; i < TLB_size; i++){
		    pthread_join(threads[i], NULL);
	    }
```

Se não estiver na TLB, procura na pagetable, e se não estiver na pagetable, procura no babcking_store.bin, faz o FIFO e o lru das páginas e da TLB, faz os calculos dos valores e dos endereços, e printa os resultados:
```c
if (is_in_tlb == 0){  // se não tiver na tlb
        
        // PAGETABLE
        for (int i = 0; i < 256; i++){    // prucura na pagetable
          if(page_table[i] == page_num){  // achou na pagetable
            frame = i;                    // pega o frame
            is_in_pagetable = 1;
            break;
          }
        }    
      }


      if (is_in_pagetable == 0 && is_in_tlb == 0){
        
        page_fault++;

        if (strcmp(argv[2],"lru") == 0){
          int biggest = 0, b_index = 0;
          for (int i = 0; i < 128; i++) {
            if (LRU_frame[i] > biggest) {         
              biggest = LRU_frame[i];
              b_index = i;
            }
          }
          frame = b_index;
          backingstorebin();        // le o backingstorebin
        }
 
        if (strcmp(argv[2],"fifo") == 0){
          frame = global_index;
          global_index++;
          if (global_index == 128){ // FIFO pagina
            global_index = 0;
          }
          backingstorebin();    // le o backingstorebin
        }
        
        if (strcmp(argv[2],"lru") == 0){
          LRU_pagetable();  // LRU pagina
        }
      }
      
      if (is_in_tlb == 0){
        if (strcmp(argv[3],"fifo") == 0){
          atualizaTLB();  // FIFO TLB
        }
        
        if (strcmp(argv[3],"lru") == 0){
          LRU_TLB();  // LRU TLB
        }
      }
      
      page_table[frame] = page_num;

      physical_ad = frame * 256 + offset;

      translated_files++;

      value = physical_mem[frame][offset];
      
      

      fprintf(fc, "Virtual address: %d Physical address: %d Value: %d\n", archive, physical_ad, value);
      
    }

    pf_rate = page_fault / translated_files;
    th_rate = (double)tlb_hit / translated_files;
    if (error == 0){
      fprintf(fc,"Number of Translated Addresses = %d\n", translated_files);
      fprintf(fc, "Page Faults = %.0f\n", page_fault);
      fprintf(fc, "Page Fault Rate = %.3f\n", pf_rate);
      fprintf(fc, "TLB Hits = %d\n",tlb_hit);
      fprintf(fc, "TLB Hit Rate = %.3f\n",th_rate);
    }
```

Funções atualizaTLB() e checkTLB(int i)
```c
void atualizaTLB(){
  
    TLB[i_tlb][0] = page_num;
    TLB[i_tlb][1] = frame;

    i_tlb++;

    if (i_tlb == 16){ // FIFO tlb
      i_tlb = 0;
    }
}

void checkTLB(int i){

  if (TLB[i][0] == page_num){ // se tiver na tlb
          frame = TLB[i][1];        // pega o frame
          is_in_tlb = 1;
          tlb_hit++;
        }
}
```
Função para ler o backingstore.bin
```c
void backingstorebin(){
    fbin = fopen("BACKING_STORE.bin","rb");

    fseek(fbin, page_num * 256, SEEK_SET);
    fread(bin_buffer, sizeof(signed char), 256, fbin);

    for (int i = 0; i < 256; i++){
      physical_mem[frame][i] = bin_buffer[i]; // coloca as instruções na memória física
    }
  
    fclose(fbin);
}
```

Funções para lru
```c
void LRU_pagetable() {

    for (int i = 0; i < 128; i++) {    
        if (i == frame) {
            LRU_frame[i] = 0;
        }
        else if (LRU_frame[i] != 1001) {
            LRU_frame[i]++;
        }
    }
}

void LRU_TLB() {
    
    int biggest = -1, b_index = 0;

    for (int i = 0; i < TLB_size; i++) {
      if (TLB_LRU_frame[i][1] > biggest) {
          biggest = TLB_LRU_frame[i][1];
          b_index = i;
      }  
    }
    TLB[b_index][0] = page_num;
    TLB[b_index][1] = frame;
  
    for (int i = 0; i < TLB_size; i++) {    
        TLB_LRU_frame[i][0] = TLB[i][1];
        if (TLB[i][1] == frame) {
            TLB_LRU_frame[i][1] = 0;
        }
        else if (TLB_LRU_frame[i][1] != 1001) {
            TLB_LRU_frame[i][1]++;
        }
    }
 
}
```
