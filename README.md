#!/bin/bash
# Modificar extensión a .sh dar permisos y ejecutar ./tester.sh
# Colores
GREEN="\033[0;32m"
RED="\033[0;31m"
YELLOW="\033[1;33m"
BLUE="\033[0;34m"
MAGENTA="\033[0;35m"
CYAN="\033[0;36m"
NC="\033[0m"

# Banner
echo -e "${GREEN}
                           
                ==                
              @@@@@@             
            @@@@  @@@@            
          @@@@  ..  @@@@          
          @@  @@@@@@  @@          
          @@ @@@@@@@@ @@          
          @@ @@    @@ @@          
           @  @    @  @           
           @@  @  @  @@           
            @   @@    @            
            @        @            
       @@@:  @ :  : @  :@@@        
  @@@@   @@@  @    @  @@@   @@@@  
 @@    @@@   @      @   @@@    @@ 
 @    @@%   @        @   %@@    @ 
      @@    @-      -@    @@      
       @@    @      @    @@       
             :@    @:             
               @  @               
               @  @         GNL Crash-Kraken by fran-byte                
${NC}"

# Configuración
BUFFER_SIZE=32
TEST_DIR="gnl_test"
mkdir -p $TEST_DIR

# Función para compilar y probar
run_test_suite() {
    local suite_name=$1
    local source_file=$2
    local header_file=$3
    local utils_file=$4
    
    echo -e "${CYAN}\n=== Probando $suite_name ===${NC}"
    
    # Verificar archivos necesarios
    if [ ! -f $source_file ]; then
        echo -e "${RED}Error: No se encontró $source_file${NC}"
        return
    fi
    
    if [ ! -f $header_file ]; then
        echo -e "${RED}Error: No se encontró $header_file${NC}"
        return
    fi
    
    # Preparar archivos de prueba
    cp $source_file $TEST_DIR/get_next_line.c
    cp $header_file $TEST_DIR/get_next_line.h
    
    # Corregir includes para archivos bonus
    if [[ "$suite_name" == *"Bonus"* ]]; then
        sed -i 's/get_next_line_bonus.h/get_next_line.h/g' $TEST_DIR/get_next_line.c 2>/dev/null
    fi
    
    # Copiar utils si existe
    if [ -f $utils_file ]; then
        cp $utils_file $TEST_DIR/get_next_line_utils.c
        if [[ "$suite_name" == *"Bonus"* ]]; then
            sed -i 's/get_next_line_bonus.h/get_next_line.h/g' $TEST_DIR/get_next_line_utils.c 2>/dev/null
        fi
    fi
    
    # Crear tester
    echo -e '#include "get_next_line.h"
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char **argv) {
    int fd;
    char *line;
    
    if (argc < 2) {
        fd = 0; // STDIN
    } else {
        fd = open(argv[1], O_RDONLY);
        if (fd == -1) {
            printf("Error opening file\\n");
            return 1;
        }
    }
    
    while ((line = get_next_line(fd)) != NULL) {
        printf("%s", line);
        free(line);
    }
    
    if (fd != 0) close(fd);
    return 0;
}' > $TEST_DIR/gnl_tester.c

    # Compilar
    echo -e "${BLUE}Compilando $suite_name...${NC}"
    gcc -Wall -Wextra -Werror -D BUFFER_SIZE=$BUFFER_SIZE $TEST_DIR/gnl_tester.c \
        $TEST_DIR/get_next_line.c $TEST_DIR/get_next_line_utils.c -o $TEST_DIR/gnl_tester 2> $TEST_DIR/compile_errors.log
    
    if [ $? -ne 0 ]; then
        echo -e "${RED}Error en la compilación:${NC}"
        cat $TEST_DIR/compile_errors.log
        return
    fi

    # Función para ejecutar pruebas individuales
    run_test() {
        local test_num=$1
        local description=$2
        local input=$3
        local expected=$4
        
        TOTAL_TESTS=$((TOTAL_TESTS + 1))
        
        printf "%b" "$input" > $TEST_DIR/test_$test_num.txt
        
        if [ "$input" == "STDIN" ]; then
            output=$(printf "%b" "$expected" | $TEST_DIR/gnl_tester 2>&1 | cat -v)
        else
            output=$($TEST_DIR/gnl_tester $TEST_DIR/test_$test_num.txt 2>&1 | cat -v)
        fi
        
        expected_output=$(printf "%b" "$expected" | cat -v)
        
        if [ "$output" == "$expected_output" ]; then
            echo -e "${GREEN}[OK]${NC} $suite_name Test $test_num: $description"
            PASSED_TESTS=$((PASSED_TESTS + 1))
        else
            echo -e "${RED}[KO]${NC} $suite_name Test $test_num: $description"
            echo -e "${YELLOW}Esperado (hex):${NC}"
            printf "%b" "$expected" | hexdump -C
            echo -e "${YELLOW}Obtenido (hex):${NC}"
            printf "%b" "$output" | hexdump -C
        fi
    }

    # Tests Básicos
    echo -e "${BLUE}\n=== Tests Básicos ===${NC}"
    
    # Test 0: Archivo vacío
    run_test 0 "Archivo vacío" "" ""
    
    # Test 1: Una línea sin newline
    run_test 1 "Una línea sin newline" "Hola mundo" "Hola mundo"
    
    # Test 2: Una línea con newline
    run_test 2 "Una línea con newline" "Hola mundo\n" "Hola mundo\n"
    
    # Test 3: Múltiples líneas
    run_test 3 "Múltiples líneas" "Línea 1\nLínea 2\nLínea 3" "Línea 1\nLínea 2\nLínea 3"
    
    # Test 4: Línea muy larga
    long_line=$(printf 'A%.0s' {1..1000})
    run_test 4 "Línea muy larga (1000 chars)" "$long_line" "$long_line"
    
    # Test 5: Solo newlines
    run_test 5 "Solo newlines" "\n\n\n\n" "\n\n\n\n"
    
    # Test 6: Archivo binario
    printf "Hola\x00mundo\n" > $TEST_DIR/test_6.txt
    output=$($TEST_DIR/gnl_tester $TEST_DIR/test_6.txt 2>&1 | cat -v)
    expected_output=$(printf "Hola\x00mundo\n" | cat -v)
    if [ "$output" == "$expected_output" ]; then
        echo -e "${GREEN}[OK]${NC} $suite_name Test 6: Archivo binario"
        PASSED_TESTS=$((PASSED_TESTS + 1))
    else
        echo -e "${RED}[KO]${NC} $suite_name Test 6: Archivo binario"
        echo -e "${YELLOW}Esperado (hex):${NC}"
        printf "Hola\x00mundo\n" | hexdump -C
        echo -e "${YELLOW}Obtenido (hex):${NC}"
        printf "%b" "$output" | hexdump -C
    fi
    TOTAL_TESTS=$((TOTAL_TESTS + 1))
    
    # Test 7: STDIN
    printf "Entrada por STDIN\n" | $TEST_DIR/gnl_tester > $TEST_DIR/test_7_output.txt
    TOTAL_TESTS=$((TOTAL_TESTS + 1))
    if [ "$(cat $TEST_DIR/test_7_output.txt)" == "Entrada por STDIN" ]; then
        echo -e "${GREEN}[OK]${NC} $suite_name Test 7: Entrada por STDIN"
        PASSED_TESTS=$((PASSED_TESTS + 1))
    else
        echo -e "${RED}[KO]${NC} $suite_name Test 7: Entrada por STDIN"
    fi
    
    # Test 8: Archivo grande
    base64 /dev/urandom | head -c 10000 > $TEST_DIR/test_8.txt
    $TEST_DIR/gnl_tester $TEST_DIR/test_8.txt > $TEST_DIR/test_8_output.txt
    TOTAL_TESTS=$((TOTAL_TESTS + 1))
    if diff $TEST_DIR/test_8.txt $TEST_DIR/test_8_output.txt > /dev/null; then
        echo -e "${GREEN}[OK]${NC} $suite_name Test 8: Archivo grande (10KB)"
        PASSED_TESTS=$((PASSED_TESTS + 1))
    else
        echo -e "${RED}[KO]${NC} $suite_name Test 8: Archivo grande (10KB)"
    fi
    
    # Test 9: Descriptor inválido
    output=$($TEST_DIR/gnl_tester /no/existe.txt 2>&1)
    TOTAL_TESTS=$((TOTAL_TESTS + 1))
    if [[ "$output" == *"Error opening file"* ]]; then
        echo -e "${GREEN}[OK]${NC} $suite_name Test 9: Descriptor inválido"
        PASSED_TESTS=$((PASSED_TESTS + 1))
    else
        echo -e "${RED}[KO]${NC} $suite_name Test 9: Descriptor inválido"
    fi
    
    # Test 10: Buffer size pequeño
    BUFFER_SIZE=1 gcc -Wall -Wextra -Werror -D BUFFER_SIZE=1 $TEST_DIR/gnl_tester.c \
        $TEST_DIR/get_next_line.c $TEST_DIR/get_next_line_utils.c -o $TEST_DIR/gnl_tester_small_buffer 2> $TEST_DIR/compile_errors.log
    TOTAL_TESTS=$((TOTAL_TESTS + 1))
    if [ $? -ne 0 ]; then
        echo -e "${RED}[KO]${NC} $suite_name Test 10: Error compilando con BUFFER_SIZE=1"
        cat $TEST_DIR/compile_errors.log
    else
        printf "Línea 1\nLínea 2\n" > $TEST_DIR/test_10.txt
        output=$($TEST_DIR/gnl_tester_small_buffer $TEST_DIR/test_10.txt 2>&1 | cat -v)
        expected_output=$(printf "Línea 1\nLínea 2\n" | cat -v)
        if [ "$output" == "$expected_output" ]; then
            echo -e "${GREEN}[OK]${NC} $suite_name Test 10: Buffer size pequeño (1 byte)"
            PASSED_TESTS=$((PASSED_TESTS + 1))
        else
            echo -e "${RED}[KO]${NC} $suite_name Test 10: Buffer size pequeño (1 byte)"
        fi
    fi
    
    # Tests Bonus adicionales
    if [[ "$suite_name" == *"Bonus"* ]]; then
        echo -e "${MAGENTA}\n=== Tests Bonus Adicionales ===${NC}"
        
        # Test 11: Múltiples descriptores simultáneos
        echo "Contenido1" > $TEST_DIR/test_11_1.txt
        echo "Contenido2" > $TEST_DIR/test_11_2.txt
        TOTAL_TESTS=$((TOTAL_TESTS + 1))
        output1=$($TEST_DIR/gnl_tester $TEST_DIR/test_11_1.txt 2>&1)
        output2=$($TEST_DIR/gnl_tester $TEST_DIR/test_11_2.txt 2>&1)
        if [ "$output1" == "Contenido1" ] && [ "$output2" == "Contenido2" ]; then
            echo -e "${GREEN}[OK]${NC} $suite_name Test 11: Múltiples descriptores simultáneos"
            PASSED_TESTS=$((PASSED_TESTS + 1))
        else
            echo -e "${RED}[KO]${NC} $suite_name Test 11: Múltiples descriptores simultáneos"
        fi
        
        # Test 12: Muchos descriptores
        echo "Test" > $TEST_DIR/test_12.txt
        ulimit -n 1024
        TOTAL_TESTS=$((TOTAL_TESTS + 1))
        for i in {1..1010}; do
            $TEST_DIR/gnl_tester $TEST_DIR/test_12.txt > /dev/null
            if [ $? -ne 0 ]; then
                echo -e "${RED}[KO]${NC} $suite_name Test 12: Muchos descriptores (falló en $i)"
                break
            fi
        done
        echo -e "${GREEN}[OK]${NC} $suite_name Test 12: Muchos descriptores (1010 archivos)"
        PASSED_TESTS=$((PASSED_TESTS + 1))
    fi
    
    # Limpieza
    rm -f $TEST_DIR/test_*.txt $TEST_DIR/*_output.txt
}

# Inicializar contadores
TOTAL_TESTS=0
PASSED_TESTS=0

# Probar parte mandatory
run_test_suite "Mandatory" "get_next_line.c" "get_next_line.h" "get_next_line_utils.c"

# Probar parte bonus (si existen los archivos)
if [ -f get_next_line_bonus.c ] && [ -f get_next_line_bonus.h ]; then
    run_test_suite "Bonus" "get_next_line_bonus.c" "get_next_line_bonus.h" "get_next_line_utils_bonus.c"
else
    echo -e "${YELLOW}\nAdvertencia: No se encontraron archivos bonus para probar${NC}"
fi

# Limpieza final
rm -rf $TEST_DIR

# Resumen
echo -e "\n${CYAN}=== Resumen Final ===${NC}"
echo -e "Total tests ejecutados: $TOTAL_TESTS"
echo -e "Tests pasados: ${GREEN}$PASSED_TESTS${NC}"
echo -e "Tests fallados: ${RED}$((TOTAL_TESTS - PASSED_TESTS))${NC}"

if [ $PASSED_TESTS -eq $TOTAL_TESTS ]; then
    echo -e "\n${GREEN}¡Todos los tests pasaron! 🎉😊🎉${NC}"
elif [ $PASSED_TESTS -gt $((TOTAL_TESTS / 2)) ]; then
    echo -e "\n${GREEN}¡La mayoría de tests pasaron! 😊${NC}"
else
    echo -e "\n${RED}Muchos tests fallaron 😞${NC}"
fi
