REPORT zz_ratio_ordres.

*---------------------------------------------------------------------*
* TYPES
*---------------------------------------------------------------------*
TYPES: BEGIN OF ty_order,
         aufnr      TYPE aufk-aufnr,
         aufnr_d    TYPE c LENGTH 12,
         qmnum      TYPE qmnum,
         notif_disp TYPE string,
         status     TYPE string,
         rowcolor   TYPE lvc_s_scol,
         ingpr      TYPE afih-ingpr, " Groupe planificateur
       END OF ty_order.

*---------------------------------------------------------------------*
* DONNÉES
*---------------------------------------------------------------------*
DATA: lt_orders      TYPE TABLE OF ty_order,
      ls_order       TYPE ty_order.

DATA: go_container_alv  TYPE REF TO cl_gui_custom_container,
      go_container_head TYPE REF TO cl_gui_custom_container,
      go_alv            TYPE REF TO cl_gui_alv_grid,
      go_text           TYPE REF TO cl_gui_textedit.

*---------------------------------------------------------------------*
* SÉLECTION DES DONNÉES
*---------------------------------------------------------------------*
START-OF-SELECTION.

  DATA: lt_aufk      TYPE TABLE OF aufk,
        ls_aufk      TYPE aufk,
        lv_status    TYPE string,
        lt_jest      TYPE TABLE OF jest,
        lv_main_stat TYPE jest-stat,
        lv_notif     TYPE qmnum,
        lv_ingpr     TYPE afih-ingpr.

  " Sélectionner les ordres du type voulu
  SELECT * FROM aufk
    WHERE auart IN ('ZPM1', 'ZPM2', 'ZPM3', 'ZPM4', 'ZPM5', 'XPM1', 'XPM2', 'XPM3', 'XPM4', 'PM01', 'PM02', 'PM03', 'PM04')
    INTO TABLE @lt_aufk.

  IF lt_aufk IS INITIAL.
    MESSAGE 'Aucun ordre trouvé !!' TYPE 'I'.
    RETURN.
  ENDIF.

  LOOP AT lt_aufk INTO ls_aufk.
    CLEAR ls_order.
    ls_order-aufnr = ls_aufk-aufnr.

    CALL FUNCTION 'CONVERSION_EXIT_ALPHA_OUTPUT'
      EXPORTING input = ls_aufk-aufnr
      IMPORTING output = ls_order-aufnr_d.

    " Récupérer la notification liée ET le groupe planificateur (via AFIH)
    SELECT SINGLE qmnum , ingpr
      FROM afih
      WHERE aufnr = @ls_aufk-aufnr
      INTO (@lv_notif, @lv_ingpr).
    IF sy-subrc = 0.
      IF lv_notif IS NOT INITIAL.
        ls_order-qmnum = lv_notif.
        ls_order-notif_disp = lv_notif.
      ELSE.
        ls_order-qmnum = ''.
        ls_order-notif_disp = 'pas de notif'.
      ENDIF.
      ls_order-ingpr = lv_ingpr.
    ELSE.
      ls_order-qmnum = ''.
      ls_order-notif_disp = 'pas de notif'.
      ls_order-ingpr = ''.
    ENDIF.

    " Récupérer le statut principal de l'ordre
    SELECT * FROM jest WHERE objnr = @ls_aufk-objnr AND inact = ''
      INTO TABLE @lt_jest.
    READ TABLE lt_jest WITH KEY stat = 'I0045'  TRANSPORTING NO FIELDS. " TECO
    IF sy-subrc = 0.
      ls_order-status = 'Clôturé'.
      ls_order-rowcolor-color-col = 3. " Jaune
      ls_order-rowcolor-color-int = 1.
    ELSE.
      READ TABLE lt_jest INDEX 1 INTO DATA(ls_jest).
      IF sy-subrc = 0.
        SELECT SINGLE txt04 FROM tj02t
          WHERE istat = @ls_jest-stat AND spras = @sy-langu
          INTO @lv_status.
        IF sy-subrc = 0.
          ls_order-status = lv_status.
        ELSE.
          ls_order-status = ls_jest-stat.
        ENDIF.
      ELSE.
        ls_order-status = 'Inconnu'.
      ENDIF.
      ls_order-rowcolor-color-col = 5. " Vert
      ls_order-rowcolor-color-int = 1.
    ENDIF.

    " Filtrer par groupe planificateur (exemple : SCP)
    IF ls_order-ingpr = ''. " <-- Change 'SCP' par ton code ou fais un IF IN TABLE si tu veux plusieurs groupes
      APPEND ls_order TO lt_orders.
    ENDIF.

  ENDLOOP.

*---------------------------------------------------------------------*
* CALCULS DIVERS POUR LES RATIOS
*---------------------------------------------------------------------*
DATA: lv_total_ord TYPE i VALUE 0,
      lv_total_notif TYPE i VALUE 0,
      lv_closed_ord TYPE i VALUE 0,
      lv_closed_notif TYPE i VALUE 0.

LOOP AT lt_orders INTO ls_order.
  lv_total_ord = lv_total_ord + 1.
  IF ls_order-qmnum IS NOT INITIAL.
    lv_total_notif = lv_total_notif + 1.
    IF ls_order-status = 'Clôturé'.
      lv_closed_notif = lv_closed_notif + 1.
    ENDIF.
  ENDIF.
  IF ls_order-status = 'Clôturé'.
    lv_closed_ord = lv_closed_ord + 1.
  ENDIF.
ENDLOOP.

DATA: lv_ratio_cloture_avis TYPE p DECIMALS 2 VALUE 0,
      lv_ratio_cloture_ord TYPE p DECIMALS 2 VALUE 0.

IF lv_total_notif > 0.
  lv_ratio_cloture_avis = ( lv_closed_notif * 100 ) / lv_total_notif.
ENDIF.
IF lv_total_ord > 0.
  lv_ratio_cloture_ord = ( lv_closed_ord * 100 ) / lv_total_ord.
ENDIF.

CALL SCREEN 0100.

*---------------------------------------------------------------------*
* ÉCRAN 0100
*---------------------------------------------------------------------*
MODULE status_0100 OUTPUT.
  IF go_container_alv IS INITIAL.

    TRY.
        CREATE OBJECT go_container_head
          EXPORTING container_name = 'CC_HEAD' no_autodef_progid_dynnr = 'X'.

        CREATE OBJECT go_container_alv
          EXPORTING container_name = 'CC_ALV' no_autodef_progid_dynnr = 'X'.

        CREATE OBJECT go_text EXPORTING parent = go_container_head.
        CALL METHOD go_text->set_readonly_mode EXPORTING readonly_mode = 1.
        CALL METHOD go_text->set_height EXPORTING height = 8.

        DATA: lv_header TYPE string,
              lv_total_ord_c TYPE string,
              lv_closed_ord_c TYPE string,
              lv_total_notif_c TYPE string,
              lv_closed_notif_c TYPE string,
              lv_ratio_cloture_avis_c TYPE string,
              lv_ratio_cloture_ord_c TYPE string.

        lv_total_ord_c = |{ lv_total_ord }|.
        lv_closed_ord_c = |{ lv_closed_ord }|.
        lv_total_notif_c = |{ lv_total_notif }|.
        lv_closed_notif_c = |{ lv_closed_notif }|.
        lv_ratio_cloture_avis_c = |{ lv_ratio_cloture_avis DECIMALS = 1 }|.
        lv_ratio_cloture_ord_c = |{ lv_ratio_cloture_ord DECIMALS = 1 }|.

        CONCATENATE
          '==============================================' cl_abap_char_utilities=>newline
          'Nombre d''ordres : ' lv_total_ord_c cl_abap_char_utilities=>newline
          'Nombre d''avis : ' lv_total_notif_c cl_abap_char_utilities=>newline
          'Ratio des ordres de travail clôturés par rapport aux avis de maintenance : '
          lv_ratio_cloture_avis_c ' %' cl_abap_char_utilities=>newline
          'Ratio des ordres de travail clôturés par rapport aux ordres créés (tous les ordres) : '
          lv_ratio_cloture_ord_c ' %' cl_abap_char_utilities=>newline
          '==============================================' cl_abap_char_utilities=>newline
          INTO lv_header.

        CALL METHOD go_text->set_textstream EXPORTING text = lv_header.

        CREATE OBJECT go_alv EXPORTING i_parent = go_container_alv.

        DATA: lt_fcat  TYPE lvc_t_fcat,
              ls_fcat  TYPE lvc_s_fcat,
              ls_layout TYPE lvc_s_layo.

        ls_layout-sel_mode   = 'A'.
        ls_layout-cwidth_opt = 'X'.
        ls_layout-grid_title = 'Liste des ordres de maintenance (groupe SCP)'.
        ls_layout-edit = ''.
        ls_layout-no_toolbar = ''.
        ls_layout-info_fname = 'ROWCOLOR'.
        ls_layout-zebra = 'X'.

        CLEAR ls_fcat.
        ls_fcat-fieldname = 'AUFNR_D'.
        ls_fcat-coltext   = 'Ordre de maintenance'.
        ls_fcat-outputlen = 15.
        APPEND ls_fcat TO lt_fcat.

        CLEAR ls_fcat.
        ls_fcat-fieldname = 'NOTIF_DISP'.
        ls_fcat-coltext   = 'Notification liée'.
        ls_fcat-outputlen = 20.
        APPEND ls_fcat TO lt_fcat.

        CLEAR ls_fcat.
        ls_fcat-fieldname = 'STATUS'.
        ls_fcat-coltext   = 'Statut'.
        ls_fcat-outputlen = 15.
        APPEND ls_fcat TO lt_fcat.

        CLEAR ls_fcat.
        ls_fcat-fieldname = 'INGPR'.
        ls_fcat-coltext   = 'Groupe Planificateur'.
        ls_fcat-outputlen = 10.
        APPEND ls_fcat TO lt_fcat.

        CALL METHOD go_alv->set_table_for_first_display
          EXPORTING is_layout = ls_layout
          CHANGING it_outtab = lt_orders it_fieldcatalog = lt_fcat.

        CALL METHOD go_alv->set_ready_for_input EXPORTING i_ready_for_input = 0.

      CATCH cx_root INTO DATA(lx_err).
        MESSAGE lx_err->get_text( ) TYPE 'E'.
    ENDTRY.
  ENDIF.
ENDMODULE.

MODULE user_command_0100 INPUT.
  CASE sy-ucomm.
    WHEN 'BACK' OR 'EXIT' OR 'CANCEL'.
      LEAVE PROGRAM.
  ENDCASE.
ENDMODULE.
