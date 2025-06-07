FLAG_TEC: EQU H800A  ; onde armazenamos a tecla lida
PTR_NOME_TXT:  EQU H8010  ; buffer temporário para nome
PTR_NUM_TXT:   EQU H8016  ; buffer para número convertido

ROBO_DESTINO: EQU 32790  ; endereço temporário (uso interno para nome)
ROBO_TAM: EQU 21

; CESAR16i - Estrutura final de entrada de nome com interrupções corretas e controle de estado

;============================
; DEFINIÇÕES
;============================
STACK:          EQU 65408
IVET:           EQU 65470
TIMDT:          EQU 65495
INTS:           EQU 65496
INTE:           EQU 65497
TECST:          EQU 65498
TECDT:          EQU 65499
VISOR:          EQU 65500
ACTIVED_ROBOTS: EQU 8000
CURRENT_ROBOT:  EQU 8002
IndiceNome:     EQU 8004
ULT_TECLA:      EQU 8006 ; ESTOU PENSANDO EM REMOVER
;ESTADO:         EQU 8008 ; remover a logica do estado
ROBOS_ATIVOS: EQU 8009 ; NOVA VARIAVEL INDICA QUANTOS ROBOS ESTAO ATIVOS

;============================
; INICIALIZAÇÃO
	MOV #-1, FLAG_TEC
        CLR, INTS
        CLR, TECST
        CLR, TECDT
; Escrever endereço da ISR em IVET (formato big endian)
        MOV #ISR, R0
        MOV R0, IVET
        MOV, #130, INTE
        MOV, #STACK, R6

        CLR, IndiceNome
        ;CLR, ESTADO  ;REMOVER
	CLR ROBOS_ATIVOS
        MOV, #-1, ULT_TECLA
        CLR, CURRENT_ROBOT ;
        JSR, R7, limpa_visor

;============================
; LOOP PRINCIPAL
;============================


main:
	JMP SALA_DE_CONTROLE
   
SALA_DE_CONTROLE:
	CMP ROBOS_ATIVOS, #0
        BEQ, sala_robos_parados
	jmp  sala_robos_ativos

sala_robos_parados:
        JSR R7, limpa_visor ; LIMPO VISOR 
        JSR R7, msg_robos_parados ;ESCREVE MENSAGEM 'ROBOS PARADOS'
;espera_tecla:
	;se tecla == 'n' ou 'N'
	;ativar robo colocando nome
	;se tecla == '1'  && ROBOS_ATIVOS !=0
	;se tecla == '2'  && ROBOS_ATIVOS !=0
	;se tecla == '3'  && ROBOS_ATIVOS !=0
	;se tecla == '4' && ROBOS_ATIVOS !=0
	;se tecla == 'r' ou 'R && ROBOS_ATIVOS !=0
	;;SE A TECLA FOR DIFERENTE DESSAS OPÇÕES CONTINUAR ESPERANDO A TECLA

espera_tecla_parado:
        MOV FLAG_TEC, R1
        CMP R1, #-1
        BEQ espera_tecla_parado

        CMP R1, #'n'
        BEQ ativa_robo
        CMP R1, #'N'
        BEQ ativa_robo

        ; Tecla inválida → ignora
        MOV #-1, FLAG_TEC
        JMP espera_tecla_parado

; COM ROBÔS ATIVOS
sala_robos_ativos:
espera_tecla_ativo:
        MOV FLAG_TEC, R1
        CMP R1, #-1
        BEQ espera_tecla_ativo

        CMP R1, #'n'
        BEQ ativa_robo
        CMP R1, #'N'
        BEQ ativa_robo

    ;    CMP R1, #'1'
    ;    BEQ trocar_robo_1
    ;    CMP R1, #'2'
     ;   BEQ trocar_robo_2
    ;    CMP R1, #'3'
    ;    BEQ trocar_robo_3
     ;   CMP R1, #'4'
    ;    BEQ trocar_robo_4

   ;     CMP R1, #'r'
    ;    BEQ desativa_robo
     ;   CMP R1, #'R'
    ;    BEQ desativa_robo

        ; comandos wasd/z viriam aqui também

        ; tecla inválida → ignora
        MOV #-1, FLAG_TEC
        ;JMP espera_tecla_ativo
	JSR R7, mostrar_robo
	JMP SALA_DE_CONTROLE
	


;ATIVA ROBO
ativa_robo:
        MOV #-1, FLAG_TEC
        JSR R7, verifica_robos
        JMP entrada_nome_v2

;============================
; ENTRADA DE NOME
;============================
entrada_nome_v2:
        JSR, R7, limpa_visor
        JSR, R7, msg_nome_robo

        MOV CURRENT_ROBOT, R3
        ADD #'1', R3
        MOV R3, VISOR+13
        MOV #':', VISOR+14

        MOV #VISOR+15, R0      ; ponteiro para escrita no visor
        ; Calcular endereço do robô atual +1 para armazenar nome
        MOV CURRENT_ROBOT, R7
        MOV #ROBO1, R1
        CLR R2
        MOV #21, R3
nome_calc_loop:
        CMP R7, #0
        BEQ nome_calc_done
        ADD R3, R2
        DEC R7
        JMP nome_calc_loop
nome_calc_done:
        ADD R2, R1
        ADD #1, R1
        MOV R1, R2             ; agora R2 aponta para nome do robô

        MOV IndiceNome, R1     ; índice inicial

loop_nome_v2:
        MOV FLAG_TEC, R4
        CMP R4, #-1
        BEQ loop_nome_v2

        CMP R4, #13
        BEQ fim_nome_v2

        CMP R1, #5
        BGE loop_nome_v2

        MOV R4, (R0)       ; escreve no visor
        MOV R4, (R2)       ; escreve na memória do robô

        INC R0             ; próximo caractere no visor
        INC R2             ; próxima posição no robô
        INC R1             ; incrementa índice
        MOV R1, IndiceNome

        MOV #-1, FLAG_TEC
        JMP loop_nome_v2

fim_nome_v2:
        MOV #-1, FLAG_TEC

        ; Ativa o robô: marca ATIVOx = 1
        MOV CURRENT_ROBOT, R0
        MOV #ROBO1, R1
        CLR R2
        MOV #21, R3
calc_robo_end:
        CMP R0, #0
        BEQ ativa_robo_ok
        ADD R3, R2
        DEC R0
        JMP calc_robo_end
ativa_robo_ok:
        ADD R2, R1        ; R1 agora aponta para o robô
        MOV #1, (R1)      ; ATIVOx = 1
        INC ROBOS_ATIVOS

        JSR R7, mostrar_robo
        JMP espera_tecla_ativo

;============================
; INTERRUPÇÃO GERAL
;============================
ISR:
        CLR, INTE  ; desativa interrupções temporariamente
        MOV, R0, -(R6)
        MOV, R1, -(R6)
        MOV, R2, -(R6)
        MOV, R3, -(R6)

        MOV, INTS, R0
        AND, #2, R0
        BEQ, isr_fim
        JSR, R7, isr_teclado
        MOV, INTS, R0
        AND, #HFFFD, R0
        MOV, R0, INTS

isr_fim:
        MOV, (R6)+, R3
        MOV, (R6)+, R2
        MOV, (R6)+, R1
        MOV, (R6)+, R0
        MOV, #130, INTE  ; reativa interrupções
        RTI

;============================
; TRATAMENTO DE TECLADO
;============================
isr_teclado:
        MOV TECDT, R0
        MOV R0, FLAG_TEC
        CLR TECST
        RTS R7


;============================
; LIMPA VISOR
;============================
limpa_visor:
        MOV, #36, R0
        MOV, #VISOR, R1
loop_l:
        MOV, #' ', (R1)
        INC, R1
        SOB, R0, loop_l
        RTS, R7

;============================
; MENSAGENS
;============================
msg_robos_parados:
        MOV, #14, R0
        MOV, #MESSAGE1, R1
        MOV, #VISOR, R2
msg1_loop:
        MOV, (R1), (R2)
        INC, R1
        INC, R2
        SOB, R0, msg1_loop
        RTS, R7

msg_nome_robo:
        MOV, #15, R0
        MOV, #MESSAGE2, R1
        MOV, #VISOR, R2
msg2_loop:
        MOV, (R1), (R2)
        INC, R1
        INC, R2
        SOB, R0, msg2_loop
        RTS, R7

;============================
; STRINGS
;============================
MESSAGE1:
    DB ' '
    DB 'R'
    DB 'O'
    DB 'B'
    DB 'O'
    DB 'S'
    DB ' '
    DB 'P'
    DB 'A'
    DB 'R'
    DB 'A'
    DB 'D'
    DB 'O'
    DB 'S'

MESSAGE2:
    DB ' '
    DB 'N'
    DB 'O'
    DB 'M'
    DB 'E'
    DB ' '
    DB 'D'
    DB 'O'
    DB ' '
    DB 'R'
    DB 'O'
    DB 'B'
    DB 'O'
    DB ' '
    DB ':'
;============================
; ATIVAR PROXIMO ROBO AO PRESSIONAR 'N'
;============================

verifica_robos:
        CMP ATIVO1, #0
        BEQ ativa_robo1
        CMP ATIVO2, #0
        BEQ ativa_robo2
        CMP ATIVO3, #0
        BEQ ativa_robo3
        CMP ATIVO4, #0
        BEQ ativa_robo4
        JMP nenhum_livre
ativa_robo1:
        MOV, #0, CURRENT_ROBOT
        RTS R7
ativa_robo2:
        MOV, #1, CURRENT_ROBOT
        RTS R7
ativa_robo3:
        MOV, #2, CURRENT_ROBOT
        RTS R7
ativa_robo4:
        MOV, #3, CURRENT_ROBOT
        RTS R7

nenhum_livre:
        JSR, R7, limpa_visor
        MOV, #22, R0
        MOV, MSG_NAOLIVRE, R1
        MOV, #VISOR, R2
loop_nao:
        MOV, (R1), (R2)
        INC, R1
        INC, R2
        SOB, R0, loop_nao
	RTS R7
inicia_nome:
	CLR IndiceNome
        MOV, CURRENT_ROBOT, R4
        MOV, #ROBO1, R5
        CLR, R6
backup_mult:
        CMP, R4, #0
        BEQ, backup_fim_mult
        ADD, #21, R6
        DEC, R4
        JMP backup_mult
backup_fim_mult:
        ADD, R6, R5
        MOV, R5, R2  ; R2 agora aponta para ROBOx
        ADD, #1, R2  ; pula o byte ATIVO
        MOV, R2, ROBO_DESTINO
        CLR, IndiceNome
        JSR, R7, limpa_visor
        JSR, R7, msg_nome_robo
        RTS R7
MSG_NAOLIVRE:
    DB 'N'
    DB 'A'
    DB 'O'
    DB ' '
    DB 'H'
    DB 'A'
    DB ' '
    DB 'R'
    DB 'O'
    DB 'B'
    DB 'O'
    DB 'S'
    DB ' '
    DB 'L'
    DB 'I'
    DB 'V'
    DB 'R'
    DB 'E'
    DB 'S'
    DB ' '
    DB ' '
mostrar_robo:
        ; Entrada: R1 = endereço base do robô (ROBO1, ROBO2, etc)
        ; Objetivo: escrever nome e textos fixos no visor
	jsr r7,limpa_visor
        ; Copiar nome para VISOR+0 até VISOR+4
        ADD #1, R1          ; pular campo ATIVO
        MOV (R1)+,65500 
        MOV (R1)+,65501
        MOV (R1)+,65502
        MOV (R1)+,65503
        MOV (R1)+,65504
	MOV (R1)+,65505	

        ; Escrever " X:" a partir de VISOR+5
        MOV #' ', VISOR+6
        MOV #'X', 65506
        MOV #':', 65507

        ; Escrever " Y:" a partir de VISOR+13
        MOV #' ', VISOR+13
        MOV #'Y', VISOR+14
        MOV #':', VISOR+15

        ; Escrever " Vx:" a partir de VISOR+21
        MOV #' ', VISOR+22
        MOV #'V', VISOR+22
        MOV #'x', VISOR+23
        MOV #':', VISOR+24

        ; Escrever " Vy:" a partir de VISOR+27
        MOV #' ', VISOR+28
        MOV #'V', VISOR+29
        MOV #'y', VISOR+30
        MOV #':', VISOR+31

        RTS R7

;============================
; DADOS DE 4 ROBÔS EM MEMÓRIA (ENDEREÇO BASE 9000)
;============================
        ORG 9000
; ROBO1 ativo por padrão
ROBO1:
ATIVO1: DB 0              ; desativado
        DB ' '            ; nome1
        DB ' '            ; nome2
        DB ' '            ; nome3
        DB ' '            ; nome4
        DB ' '            ; nome5
        DW 0              ; X
        DW 0              ; Y
        DW 0              ; VX
        DW 0              ; VY
        DB 0              ; ESTADO_ROBO

; ROBO2 desativado
ROBO2:
ATIVO2: DB 0
        DB ' '
        DB ' '
        DB ' '
        DB ' '
        DB ' '
        DW 0
        DW 0
        DW 0
        DW 0
        DB 0

; ROBO3 desativado
ROBO3:
ATIVO3: DB 0
        DB ' '
        DB ' '
        DB ' '
        DB ' '
        DB ' '
        DW 0
        DW 0
        DW 0
        DW 0
        DB 0

; ROBO4 desativado
ROBO4:
ATIVO4: DB 0
        DB ' '
        DB ' '
        DB ' '
        DB ' '
        DB ' '
        DW 0
        DW 0
        DW 0
        DW 0
        DB 0

espera_fim:
        BR espera_fim
