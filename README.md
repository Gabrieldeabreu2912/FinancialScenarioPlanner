# FinancialScenarioPlanner
A Python tool for financial statement analysis, projection, and scenario planning. Models Income Statement, Balance Sheet, and Cash Flow, incorporating detailed debt, basic deferred taxes, and goal-seeking for minimum cash under different assumption sets (e.g., base vs. pessimistic).


import pandas as pd
import numpy as np
import copy # Para copiar diccionarios de supuestos

# --- 1. Definición y Clasificación de Cuentas (Adaptar a tu Plan Contable Real) ---
# Este es un ejemplo, asegúrate que coincida con tus nombres y estructura
CLASIFICACION_CUENTAS = {
    # BALANCE GENERAL - ACTIVOS
    'Efectivo y Equivalentes': ('Balance', 'Activo', 'Corriente'),
    'Cuentas por Cobrar Comerciales': ('Balance', 'Activo', 'Corriente'),
    'Inventarios': ('Balance', 'Activo', 'Corriente'),
    'Otros Activos Corrientes': ('Balance', 'Activo', 'Corriente'), # Para balance completo
    'Propiedad, Planta y Equipo (Neto)': ('Balance', 'Activo', 'No Corriente'),
    'Impuesto Diferido Activo': ('Balance', 'Activo', 'No Corriente'), # DTA
    'Otros Activos No Corrientes': ('Balance', 'Activo', 'No Corriente'),

    # BALANCE GENERAL - PASIVOS
    'Cuentas por Pagar Comerciales': ('Balance', 'Pasivo', 'Corriente'),
    'Deuda Corto Plazo': ('Balance', 'Pasivo', 'Corriente'), # Reemplaza Obligaciones Fin CP
    'Porción Corriente Deuda LP': ('Balance', 'Pasivo', 'Corriente'), # Nuevo para detalle deuda
    'Otros Pasivos Corrientes': ('Balance', 'Pasivo', 'Corriente'), # Para balance completo
    'Deuda Largo Plazo (No Corr)': ('Balance', 'Pasivo', 'No Corriente'), # Reemplaza Obligaciones Fin LP
    'Impuesto Diferido Pasivo': ('Balance', 'Pasivo', 'No Corriente'), # DTL
    'Otros Pasivos No Corrientes': ('Balance', 'Pasivo', 'No Corriente'), # Para balance completo

    # BALANCE GENERAL - PATRIMONIO
    'Capital Social': ('Balance', 'Patrimonio', 'Capital Contribuido'),
    'Resultados Acumulados': ('Balance', 'Patrimonio', 'Ganancias Acumuladas'),
    # Resultado del Ejercicio se calcula y se integra en Resultados Acumulados en la proyección

    # ESTADO DE RESULTADOS
    'Ingresos por Ventas': ('Resultados', 'Ingresos', 'Operacionales'),
    'Costo de Ventas': ('Resultados', 'Costos', 'Costo Mercancía Vendida'), # Resta
    'Gastos de Ventas y Admin (Fijos)': ('Resultados', 'Gastos', 'Operacionales'), # Resta
    'Gastos de Ventas y Admin (Variables)': ('Resultados', 'Gastos', 'Operacionales'),# Resta
    'Depreciación': ('Resultados', 'Gastos', 'Operacionales'), # Resta (Gasto no monetario)
    'Otros Ingresos Operacionales': ('Resultados', 'Ingresos', 'Operacionales'), # Opcional
    'Otros Gastos Operacionales': ('Resultados', 'Gastos', 'Operacionales'), # Opcional, Resta
    'Ingresos Financieros': ('Resultados', 'Ingresos', 'No Operacionales'), # Opcional
    'Gastos Financieros': ('Resultados', 'Gastos', 'No Operacionales'), # Resta (Se recalculará)
    'Impuesto sobre la Renta': ('Resultados', 'Gastos', 'Impuestos') # Resta (Gasto total, se recalculará)
}

# --- 2. Ingreso de Datos (¡¡¡AJUSTA CON TUS VALORES INICIALES!!!) ---
# Asegúrate de tener saldos iniciales para TODAS las cuentas del balance necesarias
datos_cuentas = {
    # Activos
    'Efectivo y Equivalentes': 15000.00,
    'Cuentas por Cobrar Comerciales': 25000.00,
    'Inventarios': 30000.00,
    'Otros Activos Corrientes': 2000.00,
    'Propiedad, Planta y Equipo (Neto)': 150000.00,
    'Impuesto Diferido Activo': 0.00, # Poner saldo inicial si existe DTA
    'Otros Activos No Corrientes': 5000.00,
    # Pasivos
    'Cuentas por Pagar Comerciales': 18000.00,
    'Deuda Corto Plazo': 10000.00, # Saldo inicial STD (Revolver, etc.)
    'Porción Corriente Deuda LP': 8000.00, # Estimación inicial CPLTD
    'Otros Pasivos Corrientes': 3000.00,
    'Deuda Largo Plazo (No Corr)': 32000.00, # Saldo inicial LTD (Total LP anterior - CPLTD inicial)
    'Impuesto Diferido Pasivo': 4000.00, # Saldo inicial DTL (o 0 si no existe)
    'Otros Pasivos No Corrientes': 1000.00,
    # Patrimonio
    'Capital Social': 100000.00,
    'Resultados Acumulados': 42000.00, # Saldo ANTES del resultado del periodo base
    # Estado de Resultados del Periodo Base (Histórico)
    'Ingresos por Ventas': 200000.00,
    'Costo de Ventas': 120000.00,
    'Gastos de Ventas y Admin (Fijos)': 20000.00, # Estimación Fija
    'Gastos de Ventas y Admin (Variables)': 20000.00, # Estimación Variable (ej: 10% de Ventas)
    'Depreciación': 8000.00, # Depreciación del periodo base
    # Los siguientes se recalcularán en la proyección, pero se usan para obtener ratios base
    'Gastos Financieros': 5000.00, # Informativo del periodo base
    'Impuesto sobre la Renta': 7500.00, # Gasto total de impuestos del periodo base
}

# --- 3. Organización Automática en Estados Financieros (Función Base) ---
# !!! ESTA FUNCIÓN DEBE SER ROBUSTA Y ADAPTADA A TU PLAN CONTABLE !!!
# Necesita clasificar correctamente TODAS las cuentas de `datos_cuentas`
# y calcular la Utilidad Neta del periodo base para añadirla a Resultados Acumulados iniciales.
# También debe verificar que el balance base cuadre.
def organizar_estados(datos, clasificacion):
    """
    Clasifica cuentas y calcula totales básicos del periodo histórico.
    Versión simplificada - ¡REQUIERE IMPLEMENTACIÓN COMPLETA Y ADAPTADA!
    """
    balance = {'Activo': {'Corriente': {}, 'No Corriente': {}},
               'Pasivo': {'Corriente': {}, 'No Corriente': {}},
               'Patrimonio': {'Capital Contribuido': {}, 'Ganancias Acumuladas': {}}}
    resultados = {'Ingresos': {'Operacionales': {}, 'No Operacionales': {}, 'Ajuste Ingresos': {}},
                  'Costos': {'Costo Mercancía Vendida': {}},
                  'Gastos': {'Operacionales': {}, 'No Operacionales': {}, 'Impuestos': {}}}
    utilidad_neta_base = 0

    # --- Lógica de Clasificación (Iterar sobre datos.items() y clasificar) ---
    print("ADVERTENCIA: Usando lógica SIMPLIFICADA de organizar_estados.")
    # Ejemplo mínimo de llenado (REEMPLAZAR CON LÓGICA REAL)
    for cuenta, valor in datos.items():
         if cuenta in clasificacion:
             estado, seccion, subseccion = clasificacion[cuenta]
             if estado == 'Balance':
                 if seccion in balance and subseccion in balance[seccion]:
                     balance[seccion][subseccion][cuenta] = valor
             elif estado == 'Resultados':
                 if seccion in resultados and subseccion in resultados[seccion]:
                     resultados[seccion][subseccion][cuenta] = valor

    # --- Calcular Utilidad Neta Base (Ejemplo simplificado) ---
    try:
        ventas = datos.get('Ingresos por Ventas', 0)
        cogs = datos.get('Costo de Ventas', 0)
        sga_f = datos.get('Gastos de Ventas y Admin (Fijos)', 0)
        sga_v = datos.get('Gastos de Ventas y Admin (Variables)', 0)
        dep = datos.get('Depreciación', 0)
        int_exp = datos.get('Gastos Financieros', 0) # Gasto Interés Base
        tax_exp = datos.get('Impuesto sobre la Renta', 0) # Gasto Impuesto Base
        ebt_base = ventas - cogs - sga_f - sga_v - dep - int_exp
        # Usamos el impuesto reportado base para calcular la utilidad neta base
        utilidad_neta_base = ebt_base - tax_exp
        resultados['Utilidad Neta Base'] = utilidad_neta_base
    except Exception as e:
        print(f"Error calculando utilidad neta base: {e}")

    # --- Poblar totales del balance (Ejemplo simplificado) ---
    # (Asegúrate de tener la lógica completa que suma todas las subsecciones)
    total_activo = sum(v for s in balance['Activo'].values() for sd in s.values() for v in sd.values())
    total_pasivo = sum(v for s in balance['Pasivo'].values() for sd in s.values() for v in sd.values())
    capital = datos.get('Capital Social', 0)
    res_acum_inicial = datos.get('Resultados Acumulados', 0)
    # El patrimonio final del periodo base incluye la utilidad neta de ESE periodo
    total_patrimonio = capital + res_acum_inicial + utilidad_neta_base
    balance['Total Activo'] = total_activo
    balance['Total Pasivo'] = total_pasivo
    balance['Total Patrimonio'] = total_patrimonio
    balance['Total Pasivo y Patrimonio'] = total_pasivo + total_patrimonio

    # --- Verificar Cuadre Base ---
    if abs(total_activo - (total_pasivo + total_patrimonio)) > 0.01:
        print(f"\n¡¡¡ERROR!!! El Balance General BASE NO cuadra:")
        print(f"  Total Activo: {total_activo:,.2f}")
        print(f"  Total Pasivo + Patrimonio: {total_pasivo + total_patrimonio:,.2f}")
        print(f"  Diferencia: {total_activo - (total_pasivo + total_patrimonio):,.2f}")
    else:
        print("\nEl Balance General BASE cuadra correctamente.")

    # Actualizar Resultados Acumulados para el inicio de la proyección
    # El saldo inicial de Res Acum para la proyección es el final del periodo base
    balance['Patrimonio']['Ganancias Acumuladas']['Resultados Acumulados'] = res_acum_inicial + utilidad_neta_base

    return balance, resultados

# --- 4. Cálculo de Índices Financieros (Función Placeholder) ---
def calcular_indices(balance, resultados):
    """Calcula índices financieros clave. Placeholder."""
    print("ADVERTENCIA: Usando función SIMPLIFICADA de calcular_indices.")
    # Añade aquí la lógica para calcular Liquidez, Endeudamiento, Rentabilidad, etc.
    return {}

# --- 5. Cálculo de Días Operativos Base (Función Base) ---
def calcular_dias_operativos_base(balance, resultados):
    """Calcula DSO, DIO, DPO del periodo base."""
    print("ADVERTENCIA: Usando función SIMPLIFICADA de calcular_dias_operativos_base.")
    dias_operativos = {'DSO': 0, 'DIO': 0, 'DPO': 0}
    try:
        # Usar datos del diccionario 'resultados' si se pobló correctamente
        ventas_netas = resultados.get('Ventas Netas', datos_cuentas.get('Ingresos por Ventas', 1)) # Fallback a datos_cuentas
        costo_ventas = abs(datos_cuentas.get('Costo de Ventas', 1)) # Usar valor absoluto
        # Usar datos del diccionario 'balance' si se pobló correctamente
        cxc = balance.get('Activo',{}).get('Corriente',{}).get('Cuentas por Cobrar Comerciales', datos_cuentas.get('Cuentas por Cobrar Comerciales',0))
        inv = balance.get('Activo',{}).get('Corriente',{}).get('Inventarios', datos_cuentas.get('Inventarios',0))
        cxp = balance.get('Pasivo',{}).get('Corriente',{}).get('Cuentas por Pagar Comerciales', datos_cuentas.get('Cuentas por Pagar Comerciales',0))

        dias_operativos['DSO'] = (cxc / ventas_netas) * 365 if ventas_netas != 0 else 0
        dias_operativos['DIO'] = (inv / costo_ventas) * 365 if costo_ventas != 0 else 0
        dias_operativos['DPO'] = (cxp / costo_ventas) * 365 if costo_ventas != 0 else 0
    except Exception as e:
        print(f"Error calculando días base: {e}")
    return dias_operativos


# --- 6. Proyección Estados Financieros (v3 - Iterativa) ---
def proyectar_estados_financieros_v3(balance_base, resultados_base, supuestos, periodos=1, max_iteraciones_plug=10):
    """
    Proyecta EERR, Balance y Flujo de Caja con Deuda Detallada, Imp. Diferidos (simplif.),
    Supuestos 'Otros' refinados y Goal Seek para Caja Mínima usando Deuda CP (Revolver).

    Args:
        balance_base (dict): Balance histórico (organizado por organizar_estados).
        resultados_base (dict): EERR histórico (organizado por organizar_estados).
        supuestos (dict): Parámetros de proyección (ver explicación detallada).
        periodos (int): Número de periodos a proyectar.
        max_iteraciones_plug (int): Límite de iteraciones para el goal seek de caja mínima.

    Returns:
        pandas.DataFrame: DataFrame con la proyección detallada.
    """
    proyecciones = []

    # --- Saldos Iniciales (Extraer de balance_base correctamente) ---
    # (Asegúrate que organizar_estados haya poblado balance_base adecuadamente)
    saldo_caja_inicial_abs = balance_base.get('Activo', {}).get('Corriente', {}).get('Efectivo y Equivalentes', 0)
    cxc_inicial = balance_base.get('Activo', {}).get('Corriente', {}).get('Cuentas por Cobrar Comerciales', 0)
    inventario_inicial = balance_base.get('Activo', {}).get('Corriente', {}).get('Inventarios', 0)
    otros_act_corr_inicial = balance_base.get('Activo', {}).get('Corriente', {}).get('Otros Activos Corrientes', 0)
    ppe_neto_inicial = balance_base.get('Activo', {}).get('No Corriente', {}).get('Propiedad, Planta y Equipo (Neto)', 0)
    dta_inicial = balance_base.get('Activo', {}).get('No Corriente', {}).get('Impuesto Diferido Activo', 0)
    otros_act_no_corr_inicial = balance_base.get('Activo', {}).get('No Corriente', {}).get('Otros Activos No Corrientes', 0)

    cxp_inicial = balance_base.get('Pasivo', {}).get('Corriente', {}).get('Cuentas por Pagar Comerciales', 0)
    std_inicial = balance_base.get('Pasivo', {}).get('Corriente', {}).get('Deuda Corto Plazo', 0)
    cpltd_inicial = balance_base.get('Pasivo', {}).get('Corriente', {}).get('Porción Corriente Deuda LP', 0)
    otros_pas_corr_inicial = balance_base.get('Pasivo', {}).get('Corriente', {}).get('Otros Pasivos Corrientes', 0)
    ltd_no_corr_inicial = balance_base.get('Pasivo', {}).get('No Corriente', {}).get('Deuda Largo Plazo (No Corr)', 0)
    dtl_inicial = balance_base.get('Pasivo', {}).get('No Corriente', {}).get('Impuesto Diferido Pasivo', 0)
    otros_pas_no_corr_inicial = balance_base.get('Pasivo', {}).get('No Corriente', {}).get('Otros Pasivos No Corrientes', 0)

    capital_social_inicial = balance_base.get('Patrimonio', {}).get('Capital Contribuido', {}).get('Capital Social', 0)
    # ¡Importante! El resultado acumulado inicial para la proyección es el final del periodo base
    resultados_acum_inicial = balance_base.get('Patrimonio', {}).get('Ganancias Acumuladas', {}).get('Resultados Acumulados', 0)

    dtl_dta_neto_inicial = dtl_inicial - dta_inicial # Trabajaremos con el neto

    # Variables 'anteriores' para el loop
    ventas_anterior = resultados_base.get('Ventas Netas', datos_cuentas.get('Ingresos por Ventas', 0)) # Usar valor calculado si existe
    saldo_caja_anterior = saldo_caja_inicial_abs
    cxc_anterior = cxc_inicial; inventario_anterior = inventario_inicial; otros_act_corr_anterior = otros_act_corr_inicial
    ppe_neto_anterior = ppe_neto_inicial; otros_act_no_corr_anterior = otros_act_no_corr_inicial + dta_inicial # Sumamos DTA para base Otros Act NC
    cxp_anterior = cxp_inicial; std_anterior = std_inicial; cpltd_anterior = cpltd_inicial; otros_pas_corr_anterior = otros_pas_corr_inicial
    ltd_no_corr_anterior = ltd_no_corr_inicial; dtl_dta_neto_anterior = dtl_dta_neto_inicial; otros_pas_no_corr_anterior = otros_pas_no_corr_inicial + dtl_inicial # Sumamos DTL para base Otros Pas NC
    capital_social_anterior = capital_social_inicial; resultados_acum_anterior = resultados_acum_inicial

    print("\n--- Proyección Estados Financieros Detallada (v3 - Iterativa) ---")
    print("Usando Supuestos:")
    print("-" * 30)

    def get_supuesto_periodo(key, periodo_idx, default=0):
        valor = supuestos.get(key, default)
        if isinstance(valor, list):
            try: return valor[periodo_idx]
            except IndexError: return valor[-1] if valor else default # Repite último o usa default si lista vacía
        return valor

    # --- Loop Principal por Periodo ---
    for i in range(periodos):
        periodo_n = i + 1
        print(f"\nCalculando Proyección para Periodo {periodo_n}...")

        std_plug_ajuste_periodo = 0
        iteracion_plug = 0
        periodo_calculado = False

        # --- Loop Interno (Goal Seek Caja Mínima) ---
        while iteracion_plug < max_iteraciones_plug and not periodo_calculado:
            iteracion_plug += 1
            # Evitar impresión excesiva, opcional: if iteracion_plug % 5 == 1: print(f"  Iteración {iteracion_plug}...")

            # --- 1. Proyectar EERR ---
            ventas_proy = ventas_anterior * (1 + get_supuesto_periodo('crecimiento_ventas', i))
            cogs_proy = ventas_proy * get_supuesto_periodo('cogs_pct_ventas', i)
            utilidad_bruta_proy = ventas_proy - cogs_proy
            sga_fijo_proy = get_supuesto_periodo('sga_fijo', i)
            sga_variable_proy = ventas_proy * get_supuesto_periodo('sga_variable_pct_ventas', i)
            sga_total_proy = sga_fijo_proy + sga_variable_proy
            depreciacion_libro_proy = get_supuesto_periodo('depreciacion_proyectada', i)
            ebit = utilidad_bruta_proy - sga_total_proy - depreciacion_libro_proy

            tasa_interes_std = get_supuesto_periodo('interes_deuda_tasa_std', i, supuestos.get('interes_deuda_tasa_prom', 0.08))
            tasa_interes_ltd = get_supuesto_periodo('interes_deuda_tasa_ltd', i, supuestos.get('interes_deuda_tasa_prom', 0.07))
            # Usamos saldos *iniciales* del periodo (anteriores) para el interés base
            interes_std_base = max(0, std_anterior * tasa_interes_std)
            interes_ltd_base = max(0, (cpltd_anterior + ltd_no_corr_anterior) * tasa_interes_ltd)
            # Interés sobre el ajuste plug (simplificación: se asume al inicio o promedio)
            interes_plug = max(0, std_plug_ajuste_periodo * tasa_interes_std) # Ajuste simple
            intereses_proy = interes_std_base + interes_ltd_base + interes_plug

            ebt = ebit - intereses_proy

            tasa_impuestos_efectiva = get_supuesto_periodo('tasa_impuestos_efectiva', i)
            depreciacion_fiscal_proy = depreciacion_libro_proy * get_supuesto_periodo('tax_depreciacion_factor', i, 1.0)
            diferencia_temporal = depreciacion_fiscal_proy - depreciacion_libro_proy
            cambio_dtl_dta_neto = diferencia_temporal * tasa_impuestos_efectiva
            # dtl_dta_neto_proy se calcula más abajo, aquí solo el cambio

            impuesto_corriente_proy = max(0, ebt * tasa_impuestos_efectiva) if ebt > 0 else 0
            gasto_total_impuestos_proy = impuesto_corriente_proy - cambio_dtl_dta_neto
            utilidad_neta_proy = ebt - gasto_total_impuestos_proy

            # --- 2. Proyectar Activos y Pasivos no Financieros ---
            dso = get_supuesto_periodo('dso_proyectado', i); cxc_proy = (ventas_proy / 365) * dso if ventas_proy > 0 else 0
            dio = get_supuesto_periodo('dio_proyectado', i); inventario_proy = (cogs_proy / 365) * dio if cogs_proy > 0 else 0
            dpo = get_supuesto_periodo('dpo_proyectado', i); cxp_proy = (cogs_proy / 365) * dpo if cogs_proy > 0 else 0

            otros_act_corr_proy = ventas_proy * get_supuesto_periodo('pct_otros_act_corr_sobre_ventas', i, 0)
            otros_pas_corr_proy = sga_total_proy * get_supuesto_periodo('pct_otros_pas_corr_sobre_sga', i, 0)

            capex_proy = get_supuesto_periodo('capex_proyectado', i)
            ppe_neto_proy = max(0, ppe_neto_anterior + abs(capex_proy) - depreciacion_libro_proy) # No puede ser negativo
            otros_act_no_corr_sin_dta_proy = (otros_act_no_corr_anterior - dta_inicial if iteracion_plug==1 and i==0 else otros_act_no_corr_anterior) * \
                                             (1 + get_supuesto_periodo('crecimiento_otros_act_no_corr', i, 0)) # Base sin DTA
            otros_pas_no_corr_sin_dtl_proy = (otros_pas_no_corr_anterior - dtl_inicial if iteracion_plug==1 and i==0 else otros_pas_no_corr_anterior) * \
                                             (1 + get_supuesto_periodo('crecimiento_otros_pas_no_corr', i, 0)) # Base sin DTL

            # Impuesto Diferido Neto Final
            dtl_dta_neto_proy = dtl_dta_neto_anterior + cambio_dtl_dta_neto

            # --- 3. Proyectar Deuda Detallada y Patrimonio ---
            plazo_ltd = get_supuesto_periodo('plazo_nueva_deuda_lp', i, 5)
            ltd_total_inicio_periodo = cpltd_anterior + ltd_no_corr_anterior
            # CPLTD: Simplificación - Pago anual = Saldo inicial / Plazo restante. Asumimos plazo constante para simplicidad.
            cpltd_pago_estimado = max(0, ltd_total_inicio_periodo / plazo_ltd if plazo_ltd > 0 else 0)

            nueva_ltd_proy = get_supuesto_periodo('nueva_deuda_lp_proy', i, 0)
            repagos_extra_ltd_proy = abs(get_supuesto_periodo('repagos_extra_ltd_proy', i, 0))

            # Saldo LTD no corriente: Inicial Total - Pago CPLTD + Nueva LTD - Repagos Extra
            ltd_total_final_antes_reclasif = max(0, ltd_total_inicio_periodo - cpltd_pago_estimado + nueva_ltd_proy - repagos_extra_ltd_proy)
            # Reclasificar la porción corriente del *nuevo* saldo final
            cpltd_proy_final = max(0, ltd_total_final_antes_reclasif / plazo_ltd if plazo_ltd > 0 else 0)
            ltd_no_corr_proy_final = max(0, ltd_total_final_antes_reclasif - cpltd_proy_final)

            repagos_std_prog_proy = abs(get_supuesto_periodo('repagos_std_programados_proy', i, 0))
            # Saldo STD: Inicial - Repagos Programados + Ajuste Plug (iterativo)
            std_proy_final = max(0, std_anterior - repagos_std_prog_proy + std_plug_ajuste_periodo)

            nuevo_capital = get_supuesto_periodo('nuevo_capital_proy', i)
            capital_social_proy = capital_social_anterior + nuevo_capital
            dividendos_abs = abs(get_supuesto_periodo('dividendos_proy', i))
            resultados_acum_proy = resultados_acum_anterior + utilidad_neta_proy - dividendos_abs

            # --- 4. Calcular Flujo de Caja ---
            delta_cxc = cxc_proy - cxc_anterior
            delta_inventario = inventario_proy - inventario_anterior
            delta_cxp = cxp_proy - cxp_anterior
            delta_otros_act_corr = otros_act_corr_proy - otros_act_corr_anterior
            delta_otros_pas_corr = otros_pas_corr_proy - otros_pas_corr_anterior
            cambio_en_dtl_dta_para_fco = -cambio_dtl_dta_neto

            fco = utilidad_neta_proy + depreciacion_libro_proy + cambio_en_dtl_dta_para_fco \
                  - delta_cxc - delta_inventario - delta_otros_act_corr \
                  + delta_cxp + delta_otros_pas_corr

            fci = capex_proy

            fcf = (nueva_ltd_proy                 # Entrada Nueva LTD
                   - repagos_extra_ltd_proy      # Salida Extra LTD
                   - cpltd_pago_estimado         # Salida Pago CPLTD (Estimado)
                   - repagos_std_prog_proy       # Salida Pago STD Prog.
                   + std_plug_ajuste_periodo     # +/- Plug STD (Iterativo)
                   + nuevo_capital               # Entrada Capital
                   - dividendos_abs)             # Salida Dividendos

            flujo_neto_caja = fco + fci + fcf
            saldo_caja_final_calculado = saldo_caja_anterior + flujo_neto_caja

            # --- 5. Goal Seek Logic ---
            saldo_minimo_caja_req = get_supuesto_periodo('saldo_minimo_caja', i)
            if saldo_caja_final_calculado < saldo_minimo_caja_req:
                shortfall = saldo_minimo_caja_req - saldo_caja_final_calculado
                std_plug_ajuste_periodo += shortfall
                if iteracion_plug >= max_iteraciones_plug:
                    print(f"  ADVERTENCIA: Goal Seek Caja Mínima superó {max_iteraciones_plug} iteraciones.")
                    periodo_calculado = True # Forzar salida
            else:
                exceso_caja_sobre_minimo = saldo_caja_final_calculado - saldo_minimo_caja_req
                repagar_del_plug = min(exceso_caja_sobre_minimo, max(0, std_plug_ajuste_periodo))
                if repagar_del_plug > 0.01:
                     std_plug_ajuste_periodo -= repagar_del_plug
                     # Forzar una última iteración para recalcular con el repago
                     if iteracion_plug >= max_iteraciones_plug:
                         print(f"  ADVERTENCIA: Goal Seek (Repago Plug) superó {max_iteraciones_plug} iteraciones.")
                         periodo_calculado = True # Evitar bucle infinito
                     else:
                         periodo_calculado = False # Forzar una iteración más
                else:
                    periodo_calculado = True # Saldo OK, salir del while

        # --- 6. Construir Balance Proyectado Final del Periodo ---
        bs_proy_caja = saldo_caja_anterior + flujo_neto_caja # Usar flujo neto final
        bs_proy_cxc = cxc_proy
        bs_proy_inventario = inventario_proy
        bs_proy_otros_act_corr = otros_act_corr_proy
        bs_proy_total_act_corr = bs_proy_caja + bs_proy_cxc + bs_proy_inventario + bs_proy_otros_act_corr

        bs_proy_ppe_neto = ppe_neto_proy
        bs_proy_dta = max(0, -dtl_dta_neto_proy)
        bs_proy_otros_act_no_corr = otros_act_no_corr_sin_dta_proy + bs_proy_dta
        bs_proy_total_act_no_corr = bs_proy_ppe_neto + bs_proy_otros_act_no_corr

        bs_proy_total_activo = bs_proy_total_act_corr + bs_proy_total_act_no_corr

        bs_proy_cxp = cxp_proy
        bs_proy_std = std_proy_final
        bs_proy_cpltd = cpltd_proy_final
        bs_proy_otros_pas_corr = otros_pas_corr_proy
        bs_proy_total_pas_corr = bs_proy_cxp + bs_proy_std + bs_proy_cpltd + bs_proy_otros_pas_corr

        bs_proy_ltd_no_corr = ltd_no_corr_proy_final
        bs_proy_dtl = max(0, dtl_dta_neto_proy)
        bs_proy_otros_pas_no_corr = otros_pas_no_corr_sin_dtl_proy + bs_proy_dtl
        bs_proy_total_pas_no_corr = bs_proy_ltd_no_corr + bs_proy_otros_pas_no_corr

        bs_proy_total_pasivo = bs_proy_total_pas_corr + bs_proy_total_pas_no_corr

        bs_proy_capital_social = capital_social_proy
        bs_proy_resultados_acum = resultados_acum_proy
        bs_proy_total_patrimonio = bs_proy_capital_social + bs_proy_resultados_acum

        bs_proy_total_pasivo_patrimonio = bs_proy_total_pasivo + bs_proy_total_patrimonio

        diferencia_balance = bs_proy_total_activo - bs_proy_total_pasivo_patrimonio
        cuadra = abs(diferencia_balance) < 0.01 # Tolerancia

        # --- 7. Guardar Resultados del Periodo ---
        proyecciones.append({
            'Periodo': periodo_n, 'Iteraciones Plug': iteracion_plug,
            'Ventas': ventas_proy, 'EBIT': ebit, 'Intereses': -intereses_proy,
            'EBT': ebt, 'Imp Gasto Total': -gasto_total_impuestos_proy,'Utilidad Neta': utilidad_neta_proy,
            'FCO': fco, 'FCI': fci, 'STD Plug +/-': std_plug_ajuste_periodo, 'FCF': fcf,
            'Flujo Neto Caja': flujo_neto_caja, 'Saldo Caja Final': bs_proy_caja,
            'BS: Activo Total': bs_proy_total_activo, 'BS: STD': bs_proy_std, 'BS: CPLTD': bs_proy_cpltd,
            'BS: LTD No Corr': bs_proy_ltd_no_corr, 'BS: DTL/DTA Neto': dtl_dta_neto_proy,
            'BS: Pasivo Total': bs_proy_total_pasivo, 'BS: Patrimonio Total': bs_proy_total_patrimonio,
            'BS: Pasivo+Patrimonio': bs_proy_total_pasivo_patrimonio,
            'BS: Diferencia': diferencia_balance, 'BS: Cuadra': cuadra
        })

        # --- 8. Actualizar saldos 'anteriores' para el siguiente periodo ---
        ventas_anterior = ventas_proy
        saldo_caja_anterior = bs_proy_caja
        cxc_anterior = bs_proy_cxc; inventario_anterior = bs_proy_inventario; otros_act_corr_anterior = bs_proy_otros_act_corr
        ppe_neto_anterior = bs_proy_ppe_neto; otros_act_no_corr_anterior = bs_proy_otros_act_no_corr - bs_proy_dta # Ajustar base otros activos
        cxp_anterior = bs_proy_cxp; std_anterior = bs_proy_std; cpltd_anterior = bs_proy_cpltd; otros_pas_corr_anterior = bs_proy_otros_pas_corr
        ltd_no_corr_anterior = bs_proy_ltd_no_corr; dtl_dta_neto_anterior = dtl_dta_neto_proy; otros_pas_no_corr_anterior = bs_proy_otros_pas_no_corr - bs_proy_dtl # Ajustar base otros pasivos
        capital_social_anterior = bs_proy_capital_social; resultados_acum_anterior = bs_proy_resultados_acum

    return pd.DataFrame(proyecciones)


# --- Ejecución Principal ---
if __name__ == "__main__":
    # --- Organización Base, Índices Base, Días Base ---
    print("Organizando Estados Financieros Base...")
    # ¡Asegúrate que esta función esté implementada y devuelva diccionarios correctos!
    balance_actual, resultados_actuales = organizar_estados(datos_cuentas, CLASIFICACION_CUENTAS)

    print("\nCalculando Índices Base...")
    indices_calculados = calcular_indices(balance_actual, resultados_actuales) # Implementa esta función
    # ... (imprimir índices) ...

    print("\nCalculando Días Operativos Base...")
    dias_base = calcular_dias_operativos_base(balance_actual, resultados_actuales) # Implementa esta función
    print(f"  DSO Base: {dias_base.get('DSO', 0):.1f} días")
    print(f"  DIO Base: {dias_base.get('DIO', 0):.1f} días")
    print(f"  DPO Base: {dias_base.get('DPO', 0):.1f} días")


    # --- Explicación Detallada de Supuestos (v3) ---
    print("\n" + "="*40)
    print(" EXPLICACIÓN DE SUPUESTOS DE PROYECCIÓN (v3)")
    print("="*40)
    # (Incluir aquí la explicación detallada de supuestos de la respuesta anterior)
    print("""
    (Supuestos anteriores como crecimiento_ventas, cogs%, sga_fijo, sga_variable%,
     tasa_impuestos, dso, dio, dpo, depreciacion, capex, nuevo_capital, dividendos
     siguen siendo relevantes... Ver explicaciones anteriores)

    Nuevos Supuestos y Refinamientos Importantes:

    * 'pct_otros_act_corr_sobre_ventas': Proyecta 'Otros Activos Corrientes' (ej: prepagos) como % de las Ventas.
    * 'pct_otros_pas_corr_sobre_sga': Proyecta 'Otros Pasivos Corrientes' (ej: gastos acumulados) como % de los Gastos Operativos Totales (SG&A).
    * 'crecimiento_otros_act_no_corr', 'crecimiento_otros_pas_no_corr': Tasa de crecimiento simple para otras cuentas no corrientes menores (default 0%).
    * 'plazo_nueva_deuda_lp': Plazo estimado en años para la nueva deuda a largo plazo (usado para estimar CPLTD). Es una simplificación.
    * 'tax_depreciacion_factor': Multiplicador de la depreciación contable para estimar la fiscal (ej: 1.2 = 120%). Genera impuestos diferidos.
    * 'saldo_minimo_caja': Nivel mínimo de efectivo deseado. Activa el 'goal seek' con deuda CP (plug).
    * 'interes_deuda_tasa_std', 'interes_deuda_tasa_ltd': Tasas separadas para deuda CP y LP.
    * 'nueva_deuda_lp_proy': Monto específico de *nueva* Deuda a Largo Plazo planeada.
    * 'repagos_extra_ltd_proy': Pagos *adicionales* sobre principal de LTD (negativo = salida).
    * 'repagos_std_programados_proy': Pagos planeados sobre principal de STD existente (negativo = salida).
    * NOTA Deuda: CPLTD se paga automáticamente (estimado). STD actúa como 'plug' para caja mínima.
    """)
    print("="*40 + "\n")


    # --- Definir Supuestos para Escenario Base (v3) ---
    print("--- DEFINIENDO SUPUESTOS v3: ESCENARIO BASE ---")
    # !!! AJUSTA ESTOS SUPUESTOS CUIDADOSAMENTE !!!
    supuestos_base_v3 = {
        'crecimiento_ventas': [0.10, 0.08, 0.07],
        'cogs_pct_ventas': datos_cuentas.get('Costo de Ventas', 0) / datos_cuentas.get('Ingresos por Ventas', 1),
        'sga_fijo': [20000 * 1.03, 20000 * 1.03**2, 20000 * 1.03**3],
        'sga_variable_pct_ventas': datos_cuentas.get('Gastos de Ventas y Admin (Variables)', 0) / datos_cuentas.get('Ingresos por Ventas', 1),
        'tasa_impuestos_efectiva': 0.28,
        'dso_proyectado': dias_base.get('DSO', 45),
        'dio_proyectado': dias_base.get('DIO', 60),
        'dpo_proyectado': dias_base.get('DPO', 40),
        'depreciacion_proyectada': [datos_cuentas.get('Depreciación', 8000) * 1.1, datos_cuentas.get('Depreciación', 8000) * 1.2, datos_cuentas.get('Depreciación', 8000) * 1.3],
        'capex_proyectado': [-20000, -22000, -18000], # Negativo = Salida
        'nuevo_capital_proy': [0, 0, 0],
        'dividendos_proy': [-5000, -5000, -5500], # Negativo = Salida (Ajusta lógica si depende de Utilidad Neta)
        'pct_otros_act_corr_sobre_ventas': datos_cuentas.get('Otros Activos Corrientes', 0) / datos_cuentas.get('Ingresos por Ventas', 1),
        'pct_otros_pas_corr_sobre_sga': datos_cuentas.get('Otros Pasivos Corrientes', 0) / (datos_cuentas.get('Gastos de Ventas y Admin (Fijos)', 1) + datos_cuentas.get('Gastos de Ventas y Admin (Variables)', 1)),
        'crecimiento_otros_act_no_corr': 0.02,
        'crecimiento_otros_pas_no_corr': 0.02,
        'plazo_nueva_deuda_lp': 5, # Años
        'tax_depreciacion_factor': 1.2,
        'saldo_minimo_caja': 10000,
        'interes_deuda_tasa_std': 0.09,
        'interes_deuda_tasa_ltd': 0.07,
        'nueva_deuda_lp_proy': [0, 15000, 0],
        'repagos_extra_ltd_proy': [0, -2000, -2000], # Negativo
        'repagos_std_programados_proy': [-3000, -3000, -3000], # Negativo
    }

    # --- Definir Supuestos Pesimistas (v3) ---
    print("\n--- DEFINIENDO SUPUESTOS v3: ESCENARIO PESIMISTA ---")
    supuestos_pesimista_v3 = copy.deepcopy(supuestos_base_v3)
    supuestos_pesimista_v3.update({
        'crecimiento_ventas': [-0.05, 0.00, 0.02],
        'cogs_pct_ventas': supuestos_base_v3['cogs_pct_ventas'] * 1.05,
        'sga_variable_pct_ventas': supuestos_base_v3['sga_variable_pct_ventas'] * 1.03,
        'dso_proyectado': dias_base.get('DSO', 45) + 15,
        'dio_proyectado': dias_base.get('DIO', 60) + 10,
        'dpo_proyectado': dias_base.get('DPO', 40) - 5,
        'capex_proyectado': [-10000, -8000, -8000],
        'dividendos_proy': [0, 0, 0],
        'tax_depreciacion_factor': 1.0,
        'saldo_minimo_caja': 5000,
        'interes_deuda_tasa_std': 0.11,
        'interes_deuda_tasa_ltd': 0.09,
        'nueva_deuda_lp_proy': [0, 0, 0],
        'repagos_extra_ltd_proy': [0, 0, 0],
    })

    num_periodos_proyectar = 3
    max_iter_plug = 15

    # --- Ejecutar y Mostrar Proyecciones v3 ---
    print("\n" + "="*40); print(" EJECUTANDO PROYECCIÓN v3 - ESCENARIO BASE"); print("="*40)
    df_base_v3 = proyectar_estados_financieros_v3(balance_actual, resultados_actuales, supuestos_base_v3, num_periodos_proyectar, max_iter_plug)
    print("\n--- Resultados Proyección Base v3 (DataFrame) ---")
    pd.options.display.float_format = '{:,.2f}'.format
    pd.set_option('display.max_columns', None); pd.set_option('display.width', 200)
    print(df_base_v3.to_string(index=False))
    if not df_base_v3.empty and df_base_v3['BS: Cuadra'].all(): print("\nVALIDACIÓN: Balance BASE v3 cuadra.")
    else: print("\n¡¡¡ERROR!!!: Balance BASE v3 NO cuadra o DataFrame vacío.")

    print("\n" + "="*40); print(" EJECUTANDO PROYECCIÓN v3 - ESCENARIO PESIMISTA"); print("="*40)
    df_pesimista_v3 = proyectar_estados_financieros_v3(balance_actual, resultados_actuales, supuestos_pesimista_v3, num_periodos_proyectar, max_iter_plug)
    print("\n--- Resultados Proyección Pesimista v3 (DataFrame) ---")
    print(df_pesimista_v3.to_string(index=False))
    if not df_pesimista_v3.empty and df_pesimista_v3['BS: Cuadra'].all(): print("\nVALIDACIÓN: Balance PESIMISTA v3 cuadra.")
    else: print("\n¡¡¡ERROR!!!: Balance PESIMISTA v3 NO cuadra o DataFrame vacío.")

    # --- Comparación Final ---
    if not df_base_v3.empty and not df_pesimista_v3.empty:
        print("\n--- Comparación Saldo de Caja Final y STD Plug v3 ---")
        df_comparacion = pd.DataFrame({
            'Periodo': df_base_v3['Periodo'],
            'Caja Final Base': df_base_v3['Saldo Caja Final'],
            'Caja Final Pesimista': df_pesimista_v3['Saldo Caja Final'],
            'STD Plug Base': df_base_v3['STD Plug +/-'],
            'STD Plug Pesimista': df_pesimista_v3['STD Plug +/-']
        })
        print(df_comparacion.to_string(index=False))
    else:
        print("\nNo se pueden comparar resultados debido a errores en las proyecciones.")
