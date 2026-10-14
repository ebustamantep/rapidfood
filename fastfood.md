# FASTFOODPOS — Especificación del módulo Dolibarr

> **Versión objetivo:** Dolibarr **24.0.x** (verificado contra 24.0.2) con **TakePOS**.
> **Demo visual de referencia:** `index.html` de este repositorio — https://ebustamantep.github.io/rapidfood/
> **Antecedente:** módulo SabrooskiPOS (heladería), mismo patrón de botón + popup sobre TakePOS.

Este documento es la guía para construir el módulo `fastfoodpos`: qué categorías y productos
crear, qué campos adicionales (extrafields) registra el módulo, cómo entra y sale el inventario
y cómo funciona el botón **PedidosFood** dentro del POS nativo.

---

## Índice

1. [Objetivo y alcance](#1-objetivo-y-alcance)
2. [Requisitos y configuración previa de Dolibarr](#2-requisitos-y-configuración-previa-de-dolibarr)
3. [Arquitectura y estructura del módulo](#3-arquitectura-y-estructura-del-módulo)
4. [Modelo de catálogo: tipos de producto](#4-modelo-de-catálogo-tipos-de-producto)
5. [Categorías necesarias](#5-categorías-necesarias)
6. [Campos adicionales (extrafields)](#6-campos-adicionales-extrafields)
7. [Tabla propia: condiciones especiales](#7-tabla-propia-condiciones-especiales)
8. [Configuración del módulo (Setup)](#8-configuración-del-módulo-setup)
9. [Botón PedidosFood y popup de toma de pedido](#9-botón-pedidosfood-y-popup-de-toma-de-pedido)
10. [Cómo se arma la factura](#10-cómo-se-arma-la-factura)
11. [Inventario: descarga (ventas)](#11-inventario-descarga-ventas)
12. [Inventario: carga (compras, ajustes, mermas)](#12-inventario-carga-compras-ajustes-mermas)
13. [Devoluciones y anulaciones](#13-devoluciones-y-anulaciones)
14. [Catálogo de ejemplo](#14-catálogo-de-ejemplo)
15. [Permisos, idiomas y seguridad](#15-permisos-idiomas-y-seguridad)
16. [README del módulo (manual de configuración y uso)](#16-readme-del-módulo-manual-de-configuración-y-uso)
17. [Plan de construcción por fases](#17-plan-de-construcción-por-fases)
18. [Pruebas de aceptación](#18-pruebas-de-aceptación)
19. [Lecciones de SabrooskiPOS que aplican aquí](#19-lecciones-de-sabrooskipos-que-aplican-aquí)

---

## 1. Objetivo y alcance

Agregar al **TakePOS nativo** un botón **PedidosFood** que abra un popup táctil de toma de
pedidos para comida rápida (hamburguesas, perros calientes, refrescos, postres, helados). En el
popup el operador:

- elige categoría → producto;
- marca **condiciones especiales** (Sin cebolla, Sin ensalada, Salsas aparte…), que no se cobran;
- suma **adicionales** (queso, tocineta, huevo…), que se cobran;
- elige **opciones obligatorias** cuando el producto las tiene (sabor del refresco, sabores del helado);
- escribe una **nota** para cocina;
- revisa el **pedido actual** (editar / quitar / vaciar) y toca **Finalizar pedido**.

Todo se escribe en la **factura provisional del TakePOS** (`(PROV-POS<terminal>-<mesa>)`). Al
finalizar, el popup se cierra y el ticket nativo queda listo para **cobrar e imprimir** con el
flujo normal de TakePOS.

**Inventario por receta:** cada plato descuenta sus **insumos** (pan, carne, queso, cebolla…).
Si el cliente pide "Sin cebolla", la cebolla **no** se descuenta.

**Fuera de alcance v1:** pantalla de cocina (KDS), lotes/vencimientos de insumos (`productbatch`),
combos con precio especial (ver fase 4), multimoneda.

**Regla de oro:** no se modifica ningún archivo del core ni de `takepos/`. Todo entra por
hooks, triggers y endpoints propios.

---

## 2. Requisitos y configuración previa de Dolibarr

### 2.1 Módulos que deben estar activos

| Módulo | Para qué |
|---|---|
| Productos | Platos, adicionales, opciones e insumos |
| Categorías (etiquetas de producto) | Pestañas del POS, grupos de adicionales y opciones |
| Facturas de cliente | Factura provisional y definitiva |
| TakePOS | POS nativo donde vive el botón |
| Stocks | Descargo de insumos |
| Pedidos a proveedor + Recepciones *(recomendado)* | Carga de inventario por compras |
| Inventarios *(recomendado)* | Conteos físicos periódicos |

### 2.2 Constantes y opciones a fijar

| Dónde | Ajuste | Valor | Motivo |
|---|---|---|---|
| Inicio → Configuración → Módulos → Productos | **Kits / productos virtuales** (`PRODUIT_SOUSPRODUITS`) | **Sí** | Las recetas se guardan como kits nativos |
| Constante (Otras configuraciones) | `PRODUIT_SOUSPRODUITS_ALSO_ENABLE_PARENT_STOCK_MOVE` | **vacía / 0** | El plato no tiene stock propio, solo sus insumos |
| Constante | `PRODUIT_SOUSPRODUITS_ALWAYS_DISABLE_CHILDREN_STOCK_MOVE` | **vacía / 0** | Si se activa, los insumos no se descuentan |
| TakePOS → Terminal N | **Almacén** (`CASHDESK_ID_WAREHOUSE<N>`) | Almacén "Cocina" | De ahí salen los insumos |
| TakePOS → Terminal N | **No descontar stock** (`CASHDESK_NO_DECREASE_STOCK<N>`) | **No** | Si está en Sí, no hay descargo |
| TakePOS → Terminal N | Cliente por defecto (`CASHDESK_ID_THIRDPARTY<N>`) | "Cliente contado" | Requerido para crear la factura provisional |
| TakePOS | **Agrupar el mismo producto** (`TAKEPOS_GROUP_SAME_PRODUCT`) | **No** | Si está activo, un producto agregado desde la rejilla nativa se sumaría a una línea personalizada ("Sin cebolla") y mezclaría pedidos |
| Módulo Lotes/Series | `productbatch` | **Desactivado** (v1) | Con lotes, TakePOS usa otra rama de validación; los kits no admiten hijos con lote |
| Stocks | `STOCK_ALLOW_NEGATIVE_TRANSFER` o no fijar `STOCK_DISALLOW_NEGATIVE_TRANSFER` | Según política | Si se prohíbe stock negativo, una venta falla cuando falta un insumo |

> Verificado en el código de 24.0.2: `MouvementStock::_create()` **no mueve el stock del
> producto padre de un kit** (salvo `PRODUIT_SOUSPRODUITS_ALSO_ENABLE_PARENT_STOCK_MOVE`), y
> `_createSubProduct()` descuenta cada hijo con `incdec = 1` multiplicado por la cantidad
> (recursivo). TakePOS fuerza `STOCK_CALCULATE_ON_BILL = 1` al cobrar y valida la factura en el
> almacén del terminal. Por eso **el descargo de recetas lo hace el núcleo**; el módulo solo
> compensa los insumos omitidos (sección 11).

### 2.3 Almacenes sugeridos

- **Cocina** — insumos en uso; lo usa el terminal del POS.
- **Depósito** *(opcional)* — compras grandes; se transfiere a Cocina según se necesite.

---

## 3. Arquitectura y estructura del módulo

```
custom/fastfoodpos/
├── core/modules/modFastFoodPOS.class.php            ← descriptor: hooks, triggers, extrafields, permisos, constantes
├── core/triggers/interface_99_modFastFoodPOS_FastFoodPOSTriggers.class.php
│                                                    ← BILL_VALIDATE / BILL_UNVALIDATE: compensa insumos omitidos
├── class/actions_fastfoodpos.class.php              ← hooks ActionButtons (botón) + addHtmlHeader (abre el popup)
├── class/fastfoodcondicion.class.php                ← CRUD de condiciones especiales
├── class/fastfoodorder.class.php                    ← lógica de negocio: armar líneas, resolver config, omisiones
├── class/fastfoodcheck.class.php                    ← chequeos del Diagnóstico
├── class/fastfoodcatalog.class.php                  ← exportación / importación del catálogo
├── picker/index.php                                 ← contenedor del popup (inyecta window.FASTFOODPOS)
├── picker/api.php                                   ← endpoints JSON (sección 9.4)
├── js/picker.js                                     ← UI del popup (port del demo index.html)
├── css/picker.css                                   ← estilos (paleta del demo)
├── admin/setup.php                                  ← configuración general y por terminal
├── admin/condiciones.php                            ← mantenimiento de condiciones especiales
├── admin/diagnostico.php                            ← semáforo de requisitos y compatibilidad (sección 8.4)
├── admin/catalogo.php                               ← exportar / importar el catálogo en JSON (sección 8.5)
├── admin/about.php
├── lib/fastfoodpos.lib.php                          ← helpers (constantes por terminal, pestañas admin)
├── sql/llx_fastfoodpos_condicion.sql
├── sql/llx_fastfoodpos_condicion.key.sql
├── langs/es_ES/fastfoodpos.lang
├── langs/en_US/fastfoodpos.lang
├── data/catalogo-ejemplo.json                       ← catálogo de la sección 14, listo para importar
├── img/readme/                                      ← capturas del manual
└── README.md                                        ← manual paso a paso de configuración y uso (sección 16, obligatorio)
```

### 3.1 Puntos de enganche con el núcleo

| Enganche | Contexto / evento | Uso |
|---|---|---|
| Hook `ActionButtons` | `takeposfrontend` (takepos/index.php) | Agrega el botón **PedidosFood** |
| Hook `addHtmlHeader` | `takeposfrontend` (main.inc.php → top_htmlhead) | Inyecta `FastFoodPedidos()` que abre el popup |
| Trigger `BILL_VALIDATE` | Validación de factura (al cobrar en TakePOS) | Devuelve al almacén los insumos omitidos ("Sin cebolla") |
| Trigger `BILL_UNVALIDATE` | Factura vuelta a borrador | Revierte la compensación |
| Endpoint nativo `takepos/ajax/ajax.php?action=getProducts` | — | Lista los productos de cada pestaña (igual que la rejilla nativa) |
| Función JS nativa `Refresh()` del padre | — | Recarga el ticket al cerrar el popup |

Descriptor (`module_parts`):

```php
'triggers' => 1,
'hooks' => array('data' => array('takeposfrontend')),
```

### 3.2 Identificación del módulo

| Propiedad | Valor |
|---|---|
| Clase | `modFastFoodPOS` |
| `rights_class` | `fastfoodpos` |
| `numero` | **750200** (Sabrooski usa 750000; reservar un rango propio para no chocar) |
| Depende de | `modTakePos`, `modStock`, `modCategorie` |
| Dolibarr mínimo | `array(24, 0)` |
| PHP mínimo | `array(8, 1)` |

### 3.3 Portabilidad

| Qué | Cómo se garantiza |
|---|---|
| Sin cambios al core ni a `takepos/` | Solo hooks, un trigger y archivos propios en `custom/fastfoodpos/` |
| Instalación por copia | Activar el módulo crea extrafields, tabla, permisos, constantes y hooks |
| Sin IDs ni rutas fijas | Categorías elegidas en Setup; rutas con `dol_buildpath()`; tablas con `MAIN_DB_PREFIX` |
| Ajustes de Dolibarr que el módulo exige | **Diagnóstico** (8.4, grupos A–C) |
| Dependencias internas de TakePOS ante actualizaciones | **Diagnóstico** (8.4, grupo D) |
| Datos del negocio (categorías, recetas, condiciones) | **Exportar / importar catálogo** (8.5) sin IDs, por rutas y refs |
| Puesta en marcha rápida | `data/catalogo-ejemplo.json` |

---

## 4. Modelo de catálogo: tipos de producto

Cada producto del catálogo cumple **un** rol. El rol se guarda en el extrafield `ffp_rol`
(sección 6) y define cómo se muestra en el popup y cómo afecta al inventario.

| Rol | Ejemplos | Tipo Dolibarr | ¿Se vende? | Stock propio | Receta (kit) |
|---|---|---|---|---|---|
| **Plato** | Hamburguesa clásica, Perro especial, Copa de helado | Producto | Sí | No (es padre de kit) | **Sí**: sus insumos |
| **Reventa** | Agua mineral, Malta, Torta empacada | Producto | Sí | **Sí** | No |
| **Adicional** | Queso amarillo, Tocineta, Bola extra | Producto | Sí (con precio) | No | **Sí**: 1 lonja de queso, 2 tiras de tocineta… |
| **Opción** | Sabor Cola, Sabor Chocolate, Fruta Parchita | Producto | Sí (precio 0 o con recargo) | Según caso | Según caso (ver abajo) |
| **Insumo** | Pan de hamburguesa, Carne 150 g, Cebolla (kg), Lata Cola | Producto | **No** (`tosell = 0`) | **Sí** | No |

### 4.1 Cómo se modela cada familia

- **Hamburguesas / perros:** *Plato* con receta. Los adicionales y condiciones vienen de su categoría.
- **Refrescos de lata o botella con sabor a elegir:** el plato "Refresco lata" **no tiene receta
  ni stock** (rol Plato con kit vacío y `stockable_product = 0`). Cada sabor es una *Opción* con
  receta de 1 unidad del insumo correspondiente (Opción "Cola" → Insumo "Lata Cola 355 ml" × 1).
  Así el stock se lleva por sabor, que es lo que se compra.
- **Refresco sin opciones (agua, malta):** *Reventa*, modo **un toque** (se agrega sin abrir ventana).
- **Helados:** *Plato* con receta de envase (cono, vaso) y *Opciones* de sabor con receta en
  litros o kg de helado (Opción "Chocolate" → Insumo "Helado chocolate (L)" × 0,09).
  El número de bolas se controla con `ffp_opciones_min` / `ffp_opciones_max`.
- **Jugos naturales:** *Plato* con receta (vaso, azúcar) + *Opciones* de fruta con receta en kg.
- **Postres:** porciones hechas en casa → *Plato* con receta; empacados → *Reventa*.

> **Unidades:** los insumos se compran y se cuentan en su unidad real (unidad, kg, L). La
> cantidad de la receta admite decimales (`product_association.qty`). Recomendado: usar el
> campo **Unidad** del producto (módulo Unidades, `PRODUCT_USE_UNITS`) para que se vea "kg" o "L"
> en los movimientos.

---

## 5. Categorías necesarias

Categorías de **producto** (Productos → Categorías). Solo las hijas de la raíz del TakePOS se
ven como pestañas; el resto las usa el módulo internamente.

```
PedidosFood (raíz del TakePOS → TAKEPOS_ROOT_CATEGORY_ID)
├── Hamburguesas          ← pestaña
├── Perros calientes      ← pestaña
├── Refrescos             ← pestaña
├── Postres               ← pestaña
└── Helados               ← pestaña

FFP · Adicionales         (fuera de la raíz: no son pestañas)
├── Adicionales hamburguesa
├── Adicionales perro
├── Adicionales helado
└── Adicionales postre

FFP · Opciones
├── Sabores refresco
├── Sabores helado
└── Frutas jugo

FFP · Insumos             (tosell = 0; solo para orden e informes)
├── Panadería
├── Proteínas
├── Vegetales
├── Salsas
├── Lácteos
├── Bebidas (insumo)
├── Helados (insumo)
└── Desechables
```

**Reglas:**

- Un **Plato** o **Reventa** va en su pestaña (puede estar en más de una).
- Un **Adicional** va en una subcategoría de "FFP · Adicionales"; la pestaña o el producto dice
  qué subcategoría usa (sección 6 y 8).
- Una **Opción** va en una subcategoría de "FFP · Opciones".
- Los **Insumos** nunca van bajo la raíz del TakePOS (si no, se verían en la rejilla nativa).
- Los IDs **no van fijos en el código**: todo se elige en Setup (cada instalación tiene los suyos).
- El color de la categoría (campo nativo `color`) se usa como color de la pestaña en el popup.

---

## 6. Campos adicionales (extrafields)

Se crean en `modFastFoodPOS::init()` con `ExtraFields::addExtraField()` (firma de 24.0:
`attrname, label, type, pos, size, elementtype, unique, required, default, param, alwayseditable,
perms, list, help, computed, entity, langfile, enabled, …`). Todos con
`enabled = 'isModEnabled("fastfoodpos")'` y `langfile = 'fastfoodpos@fastfoodpos'`.

### 6.1 Producto (`elementtype = 'product'`)

| Código | Etiqueta | Tipo | Parámetros / valores | Visible | Uso |
|---|---|---|---|---|---|
| `ffp_rol` | Rol en PedidosFood | `select` | `plato`, `reventa`, `adicional`, `opcion`, `insumo` | Ficha y lista | Define comportamiento (sección 4). Vacío = `plato` si tiene receta, si no `reventa` |
| `ffp_modo` | Al tocarlo en el POS | `select` | `0` = según la categoría · `1` = abrir ventana · `2` = un toque | Ficha | Excepción a la configuración de la pestaña |
| `ffp_cat_adicionales` | Categoría de adicionales | `sellist` | `categorie:label:rowid::(type:=:0)` | Ficha | Vacío = la de la pestaña |
| `ffp_sin_adicionales` | Sin adicionales | `boolean` | — | Ficha | Ignora la categoría de adicionales de la pestaña |
| `ffp_max_adicionales` | Máx. unidades de adicionales | `int` | — | Ficha | 0 = sin límite |
| `ffp_cat_opciones` | Categoría de opciones | `sellist` | `categorie:label:rowid::(type:=:0)` | Ficha | Grupo obligatorio (sabor, fruta…) |
| `ffp_opciones_titulo` | Título de las opciones | `varchar(40)` | — | Ficha | "Sabor", "Sabores", "Fruta" |
| `ffp_opciones_min` | Mínimo de opciones | `int` | — | Ficha | 1 = obligatorio |
| `ffp_opciones_max` | Máximo de opciones | `int` | — | Ficha | Ej: barquilla doble = 2 |
| `ffp_sin_condiciones` | Ocultar condiciones | `boolean` | — | Ficha | Para productos que no admiten "Sin…" |
| `ffp_icono` | Ícono | `varchar(8)` | emoji | Ficha | Ícono de la tarjeta si no hay imagen |

> Parámetro `sellist` en 24.0: `tabla:campo_etiqueta:campo_clave:padre:filtro`, con el filtro en sintaxis
> universal `(campo:operador:valor)`. `(type:=:0)` limita a categorías de producto.

### 6.2 Línea de factura (`elementtype = 'facturedet'`)

Ocultos para el usuario (`list = 0`, `alwayseditable = 0`); los escribe y lee el módulo.

| Código | Tipo | Contenido | Uso |
|---|---|---|---|
| `ffp_rol` | `varchar(10)` | `principal`, `adicional`, `opcion` | Agrupar el resumen y la comanda |
| `ffp_linea_padre` | `int` | `rowid` de la línea principal | Quitar en cascada; mostrar adicionales bajo su plato |
| `ffp_grupo` | `varchar(32)` | UID del ítem del pedido (ej. `a3f9c2…`) | Enlace robusto que sobrevive al copiar líneas (nota de crédito) |
| `ffp_condiciones` | `varchar(255)` | "Sin cebolla, Sin ensalada" | Comanda e informes |
| `ffp_omitir` | `text` | JSON `[{"p":<id insumo>,"q":<cant. por unidad>}]` | Insumos que **no** se descuentan (sección 11) |
| `ffp_nota` | `varchar(120)` | Nota libre | Comanda |

> `ffp_grupo` existe porque al crear una **nota de crédito** TakePOS copia las líneas (con sus
> extrafields) pero cambian los `rowid`, así que `ffp_linea_padre` deja de ser válido en la
> copia. El agrupado se resuelve primero por `ffp_grupo`.

### 6.3 Factura (`elementtype = 'facture'`) — opcional, fase 3

| Código | Tipo | Uso |
|---|---|---|
| `ffp_tipo_pedido` | `select` (`aqui` = Comer aquí, `llevar` = Para llevar) | Comanda e informes |
| `ffp_num_pedido` | `int` | Número de pedido del día (#101, #102…) para llamar al cliente |

---

## 7. Tabla propia: condiciones especiales

Las condiciones ("Sin cebolla") se definen una vez y se asignan por pestaña. Cada condición
puede **apuntar a un insumo** para no descontarlo.

`sql/llx_fastfoodpos_condicion.sql`

```sql
CREATE TABLE llx_fastfoodpos_condicion (
  rowid           integer AUTO_INCREMENT PRIMARY KEY NOT NULL,
  entity          integer DEFAULT 1 NOT NULL,
  label           varchar(60) NOT NULL,          -- "Sin cebolla"
  fk_categorie    integer NOT NULL,              -- pestaña donde aplica (Hamburguesas)
  fk_product      integer DEFAULT NULL,          -- insumo que NO se descuenta (Cebolla kg); NULL = solo texto
  modo_omitir     varchar(8) DEFAULT 'total',    -- 'total' = toda la cantidad de la receta; 'parcial' = qty_omitir
  qty_omitir      double(24,8) DEFAULT NULL,     -- cantidad por unidad cuando modo_omitir = 'parcial'
  position        integer DEFAULT 0 NOT NULL,
  active          smallint DEFAULT 1 NOT NULL,
  datec           datetime,
  tms             timestamp DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  fk_user_creat   integer,
  fk_user_modif   integer
) ENGINE=innodb;
```

`sql/llx_fastfoodpos_condicion.key.sql`

```sql
ALTER TABLE llx_fastfoodpos_condicion ADD INDEX idx_ffpcond_cat (entity, fk_categorie, active);
ALTER TABLE llx_fastfoodpos_condicion ADD INDEX idx_ffpcond_label (entity, fk_categorie, label);
-- Sin UNIQUE en label: una condición puede afectar a varios insumos (ver reglas).
```

**Reglas:**

- Si `fk_product` es NULL, la condición solo viaja como texto (ej. "Salsas aparte", "Bien cocida").
- Si `fk_product` apunta a un insumo que **no está** en la receta del plato vendido, la omisión
  se ignora (no se puede devolver lo que no se descontó).
- `modo_omitir = 'total'` toma la cantidad de ese insumo en la receta del plato **en el momento
  de agregar la línea** y la guarda en `ffp_omitir`. Si luego se cambia la receta, el histórico
  no se altera.
- Una condición que afecta a **varios insumos** ("Sin ensalada" = lechuga + tomate) se guarda
  como **varias filas con el mismo `label`** en la misma categoría (una por insumo). El popup las
  agrupa por `label` y muestra un solo chip; al marcarlo se omiten todos sus insumos.
- La clave que viaja del popup a la API es `label` + categoría (no el `rowid`), para que el
  agrupado sea transparente.

Pantalla `admin/condiciones.php`: lista editable (categoría, condición, insumo, modo, cantidad,
orden, activo) con botón "Copiar condiciones de otra categoría".

---

## 8. Configuración del módulo (Setup)

`admin/setup.php`, con pestañas **General**, **Terminal 1…N**, **Condiciones**, **Diagnóstico**, **Catálogo**, **Acerca de**
(mismo patrón que SabrooskiPOS: valor por terminal con respaldo en el valor general).

### 8.1 Constantes

| Constante | Ámbito | Valor | Descripción |
|---|---|---|---|
| `FASTFOODPOS_TABCONFIG` | General | JSON | Configuración por pestaña (ver 8.2) |
| `FASTFOODPOS_HIDDEN_CATEGORIES[<term>]` | Terminal | `"8,12"` | Pestañas que no se muestran en ese terminal |
| `FASTFOODPOS_BUTTON_LABEL` | General | `PedidosFood` | Texto del botón |
| `FASTFOODPOS_BUTTON_COLOR` | General | `#d62828` | Color del botón |
| `FASTFOODPOS_SHOW_BCV` | General | `1` | Mostrar equivalente en Bs en el popup |
| `FASTFOODPOS_BCV_SOURCE` | General | `multicurrency` / `manual` | Tasa desde Multimoneda (VES) o valor manual |
| `FASTFOODPOS_BCV_RATE` | General | `200.00` | Tasa manual |
| `FASTFOODPOS_NOTE_MAXLEN` | General | `120` | Largo máximo de la nota |
| `FASTFOODPOS_ORDER_NUMBER` | General | `1` | Activar número de pedido del día (fase 3) |

### 8.2 Configuración por pestaña (`FASTFOODPOS_TABCONFIG`)

```json
{
  "12": {"modo": "ventana", "adicionales": 21, "max_adicionales": 0},
  "13": {"modo": "ventana", "adicionales": 22, "max_adicionales": 3},
  "14": {"modo": "toque",   "adicionales": 0},
  "15": {"modo": "ventana", "adicionales": 24},
  "16": {"modo": "ventana", "adicionales": 23}
}
```

Claves = ID de la pestaña (Hamburguesas = 12, etc.). Se edita con una tabla en Setup, no a mano.

### 8.3 Orden de prioridad de la configuración de un producto

1. Extrafields del producto (`ffp_modo`, `ffp_sin_adicionales`, `ffp_cat_adicionales`, `ffp_max_adicionales`, `ffp_cat_opciones`…).
2. Configuración de la pestaña desde la que se tocó (`FASTFOODPOS_TABCONFIG`).
3. Valores por defecto: ventana, sin adicionales, sin límite.

Las **condiciones** salen de `llx_fastfoodpos_condicion` filtradas por la pestaña, salvo que el
producto tenga `ffp_sin_condiciones = 1`.

### 8.4 Pestaña Diagnóstico (`admin/diagnostico.php`)

Revisa en un clic que la instalación cumpla todo lo que el módulo necesita. Cada chequeo da un
semáforo, un mensaje y un enlace **Corregir** a la pantalla de Dolibarr donde se arregla.

- 🟢 **OK** · 🟡 **Advertencia** (funciona, pero algo puede salir mal) · 🔴 **Error** (el inventario o el botón no funcionarán bien).
- La lógica vive en `class/fastfoodcheck.class.php` → `FastFoodCheck::runAll()` devuelve
  `[{grupo, codigo, nivel, mensaje, enlace}]`, para reutilizarla en otros lugares.
- Si hay algún 🔴, la pestaña **General** del Setup muestra un aviso arriba, y el popup
  PedidosFood muestra una franja roja **solo a administradores** ("Configuración incompleta: ver Diagnóstico").
- No bloquea el POS: avisa. Vender con un 🔴 queda a criterio del encargado.

#### A. Plataforma

| Código | Chequeo | Nivel si falla |
|---|---|---|
| `DOLI_VERSION` | `DOL_VERSION` ≥ 24.0 | 🔴 |
| `DOLI_VERSION_TESTED` | Versión mayor > 24 (no probada todavía con este módulo) | 🟡 |
| `PHP_VERSION` | PHP ≥ 8.1 | 🔴 |
| `MODULES` | Activos: `product`, `categorie`, `facture`, `stock`, `takepos` | 🔴 |
| `NO_BATCH` | `productbatch` desactivado | 🔴 |
| `KITS` | `PRODUIT_SOUSPRODUITS` = 1 | 🔴 |
| `KIT_PARENT_MOVE` | `PRODUIT_SOUSPRODUITS_ALSO_ENABLE_PARENT_STOCK_MOVE` vacío | 🔴 |
| `KIT_CHILD_MOVE` | `PRODUIT_SOUSPRODUITS_ALWAYS_DISABLE_CHILDREN_STOCK_MOVE` vacío | 🔴 |
| `GROUP_SAME` | `TAKEPOS_GROUP_SAME_PRODUCT` vacío | 🔴 |
| `NEGATIVE_STOCK` | `STOCK_DISALLOW_NEGATIVE_TRANSFER` activo → una venta puede fallar si falta un insumo | 🟡 |

#### B. Instalación del módulo

| Código | Chequeo | Nivel |
|---|---|---|
| `HOOKS_CONST` | `MAIN_MODULE_FASTFOODPOS_HOOKS` contiene `takeposfrontend` | 🔴 (solución: desactivar y reactivar) |
| `TRIGGERS_CONST` | `MAIN_MODULE_FASTFOODPOS_TRIGGERS` = 1 y existe el archivo del trigger | 🔴 |
| `EXTRAFIELDS` | Existen los extrafields `ffp_*` de `product`, `facturedet` (y `facture` si fase 3) | 🔴 |
| `TABLE` | Existe `llx_fastfoodpos_condicion` | 🔴 |
| `PERMS` | Al menos un usuario o grupo activo tiene `fastfoodpos->use` | 🟡 |

#### C. TakePOS y terminales

Para cada terminal `N` de 1 a `TAKEPOS_NUM_TERMINALS`:

| Código | Chequeo | Nivel |
|---|---|---|
| `ROOT_CAT` | `TAKEPOS_ROOT_CATEGORY_ID` definida y con al menos una hija | 🔴 |
| `TERM_N_CUSTOMER` | `CASHDESK_ID_THIRDPARTYN` > 0 y el tercero existe | 🔴 |
| `TERM_N_WAREHOUSE` | `CASHDESK_ID_WAREHOUSEN` > 0 y el almacén existe y está abierto | 🔴 |
| `TERM_N_DECREASE` | `CASHDESK_NO_DECREASE_STOCKN` ≠ 1 | 🔴 |
| `BCV_RATE` | Si `FASTFOODPOS_BCV_SOURCE = multicurrency`: existe tasa de VES con fecha de hoy | 🟡 |

#### D. Compatibilidad con TakePOS (protege ante actualizaciones de Dolibarr)

El módulo usa piezas internas de TakePOS que no son una API pública. Este grupo lee los
archivos nativos (solo lectura, `file_get_contents`) y confirma que siguen ahí. Si una
actualización de Dolibarr cambia alguna, el diagnóstico lo dice **antes** de que falle en caja.

| Código | Archivo | Busca | Para qué lo usa el módulo |
|---|---|---|---|
| `TP_HOOK_BUTTONS` | `takepos/index.php` | `executeHooks('ActionButtons'` | Botón PedidosFood |
| `TP_HOOK_CONTEXT` | `takepos/index.php` | `initHooks(array('takeposfrontend'))` | Contexto del hook |
| `TP_JS_REFRESH` | `takepos/index.php` | `function Refresh(` | Recargar el ticket al cerrar el popup |
| `TP_JS_PLACE` | `takepos/index.php` | `var place=` | Mesa actual |
| `TP_PROV_REF` | `takepos/invoice.php` | `(PROV-POS` | Encontrar la factura provisional |
| `TP_GETPRODUCTS` | `takepos/ajax/ajax.php` | `action == 'getProducts'` | Rejilla de productos |
| `TP_STOCK_ON_PAY` | `takepos/invoice.php` | `CASHDESK_ID_WAREHOUSE` y `->validate($user` | Descargo al cobrar |
| `CORE_KIT_PARENT` | `product/stock/class/mouvementstock.class.php` | `PRODUIT_SOUSPRODUITS_ALSO_ENABLE_PARENT_STOCK_MOVE` | El kit no mueve stock del padre |

Todos 🔴 si no se encuentran. Las cadenas buscadas se guardan en un arreglo en la clase,
con un comentario de la versión donde se verificaron (24.0.2).

#### E. Catálogo

| Código | Chequeo | Nivel |
|---|---|---|
| `CAT_INSUMO_IN_ROOT` | Ningún producto con rol *insumo* cuelga de la raíz del TakePOS | 🔴 |
| `CAT_INSUMO_TOSELL` | Insumos con "En venta = Sí" | 🟡 |
| `CAT_PLATO_NO_KIT` | Producto con rol *plato* sin receta y con stock gestionado (se descontaría el plato, no sus insumos) | 🟡 |
| `CAT_KIT_INCDEC` | Componente de receta con `incdec = 0` (no se descontará) | 🔴 |
| `CAT_KIT_SERVICE_CHILD` | Componente que es servicio o tiene "Gestión de stocks = No" | 🟡 |
| `CAT_TAB_EXTRAS` | Pestaña con categoría de adicionales inexistente o vacía | 🔴 |
| `CAT_OPTIONS` | Producto con `ffp_opciones_min` > 0 sin categoría de opciones, o con categoría vacía | 🔴 |
| `CAT_OPTION_NO_STOCK` | Opción sin receta ni stock (no descuenta nada; válido solo si es intencional) | 🟡 |
| `CAT_COND_INSUMO` | Condición que apunta a un insumo que no está en ninguna receta de su pestaña | 🟡 |
| `CAT_PRICE_ZERO` | Plato o adicional con precio 0 | 🟡 |

El resultado del grupo E lista los productos afectados con enlace a su ficha.

#### Cuándo correr el diagnóstico

- Al terminar la instalación y la configuración (paso obligatorio del README).
- **Después de cada actualización de Dolibarr** (grupo D).
- Después de importar un catálogo (8.5): se ejecuta solo al terminar la importación.

### 8.5 Pestaña Catálogo: exportar e importar (`admin/catalogo.php`)

Permite llevar **toda la configuración del negocio** de una instalación a otra (de pruebas a
producción, de un local a una sucursal, o como respaldo antes de cambios grandes) en un solo
archivo JSON. La lógica vive en `class/fastfoodcatalog.class.php`.

#### Qué se exporta y qué no

| Se exporta | No se exporta (cada instalación tiene lo suyo) |
|---|---|
| Árbol de categorías usadas por el módulo (pestañas, adicionales, opciones, insumos) con color y descripción | Terceros, usuarios, permisos |
| Productos con rol `ffp_*`: ref, nombre, descripción, tipo, precio, base de precio, IVA, en venta / en compra, gestión de stocks, unidad, stock de alerta, extrafields `ffp_*`, categorías | Almacenes y terminales (`CASHDESK_*`) |
| Recetas (kits): padre, componente, cantidad, `incdec` | **Existencias** (stock): se cargan con inventario o importación nativa (README) |
| Condiciones especiales | Ventas, facturas, movimientos |
| Configuración por pestaña (`FASTFOODPOS_TABCONFIG`) y ajustes generales del módulo (botón, color, nota, BCV manual) | Imágenes de productos (fase 5: exportación en ZIP) |

#### Regla clave: nada se referencia por ID

Los `rowid` cambian entre instalaciones. En el archivo:

- las **categorías** se identifican por su **ruta** (`"PedidosFood/Hamburguesas"`, `"FFP · Insumos/Vegetales"`);
- los **productos** por su **ref**;
- las **recetas** por ref padre + ref componente;
- las **condiciones** por ruta de pestaña + texto + ref del insumo;
- las unidades por su **código** de `c_units` (`KG`, `L`, `P`) y el IVA por su **tasa**.

Al importar, el módulo traduce rutas y refs a los IDs de la instalación destino, y reescribe
`FASTFOODPOS_TABCONFIG` y los extrafields `ffp_cat_*` con los IDs nuevos.

#### Formato del archivo

`fastfoodpos-catalogo-<AAAAMMDD-HHMM>.json`

```json
{
  "schema": "fastfoodpos-catalog",
  "schema_version": 1,
  "module_version": "1.0",
  "dolibarr_version": "24.0.2",
  "exported_at": "2026-10-14T16:30:00-04:00",
  "currency": "USD",
  "root": "PedidosFood",
  "categories": [
    {"path": "PedidosFood", "color": "", "description": ""},
    {"path": "PedidosFood/Hamburguesas", "color": "d62828", "description": ""},
    {"path": "FFP · Adicionales/Adicionales hamburguesa"},
    {"path": "FFP · Insumos/Vegetales"}
  ],
  "products": [
    {
      "ref": "INS-CEBOLLA", "label": "Cebolla", "type": 0,
      "tosell": 0, "tobuy": 1, "stockable": 1, "unit": "KG", "stock_alert": 2,
      "price": 0, "price_base_type": "HT", "tva_tx": 16,
      "categories": ["FFP · Insumos/Vegetales"],
      "ffp": {"rol": "insumo"}
    },
    {
      "ref": "HAMB-CLASICA", "label": "Hamburguesa clásica", "type": 0,
      "tosell": 1, "tobuy": 0, "stockable": 1,
      "price": 5.00, "price_base_type": "TTC", "tva_tx": 16,
      "categories": ["PedidosFood/Hamburguesas"],
      "ffp": {"rol": "plato", "modo": 0, "cat_adicionales": null, "max_adicionales": 0, "icono": "🍔"}
    }
  ],
  "recipes": [
    {"parent": "HAMB-CLASICA", "child": "INS-PAN-HAMB", "qty": 1, "incdec": 1},
    {"parent": "HAMB-CLASICA", "child": "INS-CEBOLLA", "qty": 0.03, "incdec": 1}
  ],
  "conditions": [
    {"tab": "PedidosFood/Hamburguesas", "label": "Sin cebolla", "insumo": "INS-CEBOLLA", "modo": "total", "qty": null, "position": 10, "active": 1},
    {"tab": "PedidosFood/Hamburguesas", "label": "Salsas aparte", "insumo": null, "position": 70, "active": 1}
  ],
  "tabs": [
    {"path": "PedidosFood/Hamburguesas", "modo": "ventana", "adicionales": "FFP · Adicionales/Adicionales hamburguesa", "max_adicionales": 0}
  ],
  "settings": {
    "FASTFOODPOS_BUTTON_LABEL": "PedidosFood",
    "FASTFOODPOS_BUTTON_COLOR": "#d62828",
    "FASTFOODPOS_NOTE_MAXLEN": "120"
  }
}
```

#### Exportar

1. **Setup → Catálogo → Exportar**.
2. Opciones: incluir precios (sí/no), incluir productos inactivos (sí/no).
3. Descarga el JSON. Se exportan solo productos con `ffp_rol` definido y las categorías bajo la
   raíz del TakePOS y las elegidas en la configuración (adicionales, opciones, insumos).

#### Importar

1. **Setup → Catálogo → Importar** → elegir el archivo.
2. Elegir el **modo**:
   - **Simular** *(por defecto)*: no escribe nada; muestra el informe de lo que haría.
   - **Solo crear lo nuevo**: crea lo que no existe; no toca lo existente.
   - **Crear y actualizar**: además actualiza productos existentes (misma ref), **reemplaza la
     receta completa** de cada plato importado y sincroniza condiciones de las pestañas importadas.
3. Opciones: **no tocar precios** (para sucursales con precios propios), **usar como raíz del
   TakePOS** la categoría `root` del archivo.
4. Resultado: informe por tipo (creados / actualizados / sin cambios / errores) y, al final, el
   **Diagnóstico** completo.

Reglas de la importación:

- **Todo o nada:** una sola transacción; si hay un error, no queda nada a medias.
- **Validación previa:** `schema` y `schema_version` conocidos; refs no vacías y únicas; cada
  receta y condición apunta a productos del archivo o ya existentes; tasa de IVA válida para el
  país de la empresa (`c_tva`); unidad existente (`c_units`); sin recetas circulares.
- **Moneda distinta** a la de la instalación → advertencia y sugerencia de "no tocar precios".
- **Categorías:** se crean por ruta (tipo producto) si no existen; nunca se borran.
- **Productos:** nunca se borran ni se desactivan los que no están en el archivo.
- Usa las clases nativas (`Categorie::create`, `Product::create/update`, `Product::add_sousproduit`,
  `ExtraFields` vía `array_options`, `FastFoodCondicion::create`) para respetar triggers y validaciones del núcleo; nada de SQL directo para crear productos.
- Límite: 5 MB por archivo; permiso `fastfoodpos->config` y usuario administrador.

#### Catálogo de ejemplo incluido

El módulo trae `data/catalogo-ejemplo.json` con el catálogo de la sección 14 (insumos, platos,
recetas, adicionales, opciones y condiciones). En una instalación nueva se importa en modo
**Solo crear lo nuevo** para tener un POS funcionando en minutos y luego se ajustan precios y recetas.

---

## 9. Botón PedidosFood y popup de toma de pedido

### 9.1 Botón (hook `ActionButtons`)

`class/actions_fastfoodpos.class.php`

```php
public function ActionButtons($parameters, &$object, &$action, $hookmanager)
{
    global $user, $langs;
    if (($parameters['currentcontext'] ?? '') !== 'takeposfrontend') return 0;
    if (!$user->hasRight('takepos', 'run') || !$user->hasRight('fastfoodpos', 'use')) return 0;

    $label = getDolGlobalString('FASTFOODPOS_BUTTON_LABEL', 'PedidosFood');
    $color = getDolGlobalString('FASTFOODPOS_BUTTON_COLOR', '#d62828');
    // takepos/index.php hace foreach ($resArray as $butmenu): cada elemento debe ser un ARRAY de botones.
    $this->results[] = array(array(
        'title'  => '<span class="fa fa-hamburger paddingrightonly" style="font-size:1.5em"></span><div class="trunc">'.dol_escape_htmltag($label).'</div>',
        'action' => 'FastFoodPedidos();',
        'style'  => 'background:'.$color.';color:#fff;border-color:'.$color.';',
    ));
    return 0;   // 0 = conservar los botones nativos y sumar el nuestro
}
```

### 9.2 Apertura del popup (hook `addHtmlHeader`)

```php
public function addHtmlHeader($parameters, &$object, &$action, $hookmanager)
{
    if (($parameters['currentcontext'] ?? '') !== 'takeposfrontend') return 0;
    $url = dol_buildpath('/fastfoodpos/picker/index.php', 1);
    $this->resprints = '<script>
    function FastFoodPedidos() {
        $.colorbox({href: '.json_encode($url).' + "?place=" + encodeURIComponent(place),
                    width: "96%", height: "94%", iframe: true, transition: "none",
                    title: "PedidosFood",
                    onClosed: function () { if (typeof Refresh === "function") Refresh(); }});
    }</script>';
    return 0;
}
```

- `place` es la variable global del TakePOS (mesa / lugar).
- Al cerrar el popup (por "Finalizar pedido", "Cerrar" o la X) se llama a **`Refresh()`**, la
  función nativa que recarga `#poslines` con `invoice.php?place=…`, sin recargar toda la página.

### 9.3 Popup (`picker/index.php` + `js/picker.js`)

- `top_htmlhead()` con rutas **relativas** (`/fastfoodpos/css/picker.css`), nunca con `DOL_URL_ROOT` delante.
- Inyecta `window.FASTFOODPOS = {api, base, place, token, socid, currency, bcv, i18n}` **antes** de cargar `picker.js` (al final del `<body>`).
- `picker.js` es el **port del demo** `index.html`: mismas pantallas, mismos estados, pero los
  datos salen de la API y el pedido vive en la factura provisional (fuente de verdad), no en memoria.
- Cache-buster `?v=<DOL_VERSION>-<filemtime>` en CSS y JS.

| Pantalla del demo | Dato en Dolibarr |
|---|---|
| Pestañas | `getData.categories` |
| Tarjetas | `takepos/ajax/ajax.php?action=getProducts&category=<id>&tosell=1` (endpoint nativo) |
| Insignia "1 toque" | `ffp_modo = 2` o pestaña con `modo = toque` |
| Opciones obligatorias | `getProduct.options` |
| Condiciones especiales | `getProduct.conditions` |
| Adicionales | `getProduct.extras` |
| Pedido actual | `getCart` |
| Comer aquí / Para llevar | extrafield de factura `ffp_tipo_pedido` (fase 3) |

### 9.4 Endpoints (`picker/api.php`)

Todos devuelven JSON y exigen `takepos/run` + `fastfoodpos/use`. Las acciones que escriben
exigen **POST** y `token` válido (CSRF de Dolibarr). Terminal = `$_SESSION['takeposterminal']`.

| Acción | Método | Parámetros | Respuesta |
|---|---|---|---|
| `getData` | GET | `place` | `{categories:[{id,label,color,modo}], bcv:{rate}}` |
| `getProduct` | GET | `idproduct`, `catid` | `{product:{id,label,description,price_ttc,price_formated,icono}, modo, options:{title,min,max,items:[{id,label,price_ttc}]}, conditions:[{key,label}], extras:[{id,label,price_ttc}], max_extras}` |
| `addItem` | POST | `idproduct`, `catid`, `qty`, `options` (JSON ids), `conditions` (JSON keys), `extras` (JSON `[{id,qty}]` por unidad), `note` | `{ok, invoiceid, lineid}` |
| `updateItem` | POST | `grupo` + los mismos de `addItem` | Reemplaza el ítem (borra sus líneas y lo vuelve a crear en la misma transacción) |
| `getCart` | GET | `place` | `{invoiceid, items:[{grupo, lineid, qty, label, total_ttc, options[], conditions[], extras:[{label,qty,total_ttc}], note, editable}], itemcount, total_ttc, total_formated, total_bs}` |
| `removeItem` | POST | `grupo` | Borra la línea principal y todas las de su grupo |
| `clearCart` | POST | `place` | Borra todas las líneas de la factura provisional |

Validaciones del servidor (no confiar en el navegador):

- El producto existe, está activo y en venta.
- Cada **opción** pertenece a la categoría de opciones del producto y se respeta `min`/`max`.
- Cada **adicional** pertenece a la categoría de adicionales resuelta y se respeta `max_adicionales`.
- Cada **condición** pertenece a la pestaña y está activa.
- La factura provisional está en **borrador** (`STATUS_DRAFT`); si no, `409`.
- `qty` entre 1 y 999; nota sin HTML y recortada a `FASTFOODPOS_NOTE_MAXLEN`.

---

## 10. Cómo se arma la factura

### 10.1 Factura provisional

Misma que usa TakePOS: ref `(PROV-POS<terminal>-<place>)`. Si no existe, el módulo la crea
igual que `takepos/invoice.php`: `socid = CASHDESK_ID_THIRDPARTY<term>`,
`module_source = 'takepos'`, `pos_source = <term>`, `entity` de la sesión, y luego
`UPDATE … SET ref = '(PROV-POS<term>-<place>)'`.

### 10.2 Líneas que genera un ítem

Ejemplo: **2 × Hamburguesa clásica**, *Sin cebolla*, + Queso amarillo ×1, + Tocineta ×1,
nota "salsa de ajo aparte".

| # | Producto | Cant. | Precio u. | Descripción de la línea (ticket / comanda) | `ffp_rol` | `ffp_linea_padre` | `ffp_omitir` |
|---|---|---|---|---|---|---|---|
| 1 | Hamburguesa clásica | 2 | 5,00 | `Sin cebolla \| Con: Queso amarillo, Tocineta \| Nota: salsa de ajo aparte` | principal | — | `[{"p":<Cebolla>,"q":0.03}]` |
| 2 | Queso amarillo (adicional) | 2 | 0,75 | `↳ Adicional de Hamburguesa clásica` | adicional | 1 | — |
| 3 | Tocineta (adicional) | 2 | 1,00 | `↳ Adicional de Hamburguesa clásica` | adicional | 1 | — |

Las tres líneas comparten el mismo `ffp_grupo`.

Reglas:

- **Cantidad de los adicionales y opciones = cantidad por unidad × cantidad del plato.**
- Las **opciones** se agregan como líneas propias (`ffp_rol = opcion`) a su precio (0 o recargo).
  Ej. "Refresco lata" + línea "Cola" a 0,00. Así su receta descuenta la lata correcta.
- El **precio** se calcula como lo hace TakePOS: `Product::getSellPrice($mysoc, $cliente)` →
  `pu_ht`, `pu_ttc`, `tva_tx`, `price_base_type`, impuestos locales con `get_localtax()` y
  `remise_percent` del cliente.
- Todas las líneas de un ítem se insertan en **una transacción** (`$db->begin()` / `commit()`);
  si una falla, no queda nada a medias.
- Se usa `Facture::addline()` (firma de 24.0: `desc, pu_ht, qty, txtva, localtax1, localtax2,
  fk_product, remise_percent, date_start, date_end, fk_code_ventilation, info_bits,
  fk_remise_except, price_base_type, pu_ttc, type, rang, special_code, origin, origin_id,
  fk_parent_line, fk_fournprice, pa_ht, label, array_options, …`) pasando los extrafields de
  línea en `array_options` (`options_ffp_rol`, `options_ffp_grupo`, …).
- **No** se usa `fk_parent_line` nativo (es para líneas de título/subtotal) — el enlace va en el extrafield.

### 10.3 Cálculo de `ffp_omitir`

Al agregar el ítem:

```
para cada condición marcada con fk_product = insumo:
    q_receta = cantidad del insumo en la receta del plato (product_association, incdec = 1)
               (si el insumo está en un sub-kit, multiplicar por la cantidad del sub-kit)
    si q_receta == 0 → ignorar (el insumo no está en la receta)
    q = (modo_omitir == 'total') ? q_receta : min(qty_omitir, q_receta)
    acumular {p: insumo, q: q}      ← cantidad POR UNIDAD del plato
guardar JSON en options_ffp_omitir de la línea principal
```

### 10.4 Finalizar pedido

1. El popup llama `parent.$.colorbox.close()`.
2. `onClosed` ejecuta `Refresh()` del TakePOS → el ticket muestra las líneas.
3. El cajero cobra con **Pagar** nativo (`takepos/pay.php`) → TakePOS valida la factura,
   **descarga el stock** (sección 11) y registra el pago.
4. Impresión nativa del ticket. La descripción de cada línea lleva las condiciones, opciones y
   nota, así que también sirve de comanda.

---

## 11. Inventario: descarga (ventas)

### 11.1 Qué hace el núcleo (sin código del módulo)

Al cobrar, TakePOS (24.0) valida la factura con `STOCK_CALCULATE_ON_BILL = 1` en el almacén
`CASHDESK_ID_WAREHOUSE<term>` (salvo `CASHDESK_NO_DECREASE_STOCK<term> = 1`). Para cada línea:

- **Reventa** (producto simple con stock) → sale del almacén la cantidad vendida.
- **Plato / Adicional / Opción con receta** (kit) → el padre **no** mueve stock; cada insumo
  con `incdec = 1` sale `cantidad vendida × cantidad en receta` (recursivo para sub-kits).
- **Producto con `stockable_product = 0`** o servicio → no mueve stock.

Etiqueta del movimiento: "Factura XXXX validada", origen = la factura.

> **Importante:** en la ficha de cada componente del kit, la casilla **"Incrementar/disminuir
> stock del producto hijo"** (`incdec`) debe estar **marcada**. Si no, ese insumo no se descuenta.

### 11.2 Qué hace el módulo: compensar los insumos omitidos

Trigger `BILL_VALIDATE` (`interface_99_modFastFoodPOS_FastFoodPOSTriggers`):

```
si factura.module_source != 'takepos' → salir
para cada línea con options_ffp_omitir no vacío:
    para cada {p, q} en ffp_omitir:
        cant = q × línea.qty
        buscar el movimiento nativo de ese insumo para esta factura:
            SELECT fk_entrepot, SUM(value) FROM llx_stock_mouvement
            WHERE fk_product = p AND fk_origin = factura.id AND origintype = 'facture'
              AND (inventorycode IS NULL OR inventorycode NOT LIKE 'FFP-%')
        si no hay movimiento → no hubo descargo (stock desactivado) → no compensar
        si ya existe un movimiento con inventorycode = 'FFP-<factura.id>-<línea.id>-<p>' → no repetir
        si factura.type == CREDIT_NOTE:
            MouvementStock::livraison(p, almacén, cant, 0, "Sin <insumo> – NC <ref>", inventorycode)
        si no:
            MouvementStock::reception(p, almacén, cant, 0, "Omitido: <condición> – Factura <ref>", inventorycode)
```

- Precio 0 en la compensación para **no alterar el PMP** (costo promedio), igual que hace el núcleo con los hijos de kits.
- `setOrigin('facture', $factura->id)` para que el movimiento aparezca en la ficha de la factura.
- `inventorycode = 'FFP-<idfactura>-<idlinea>-<idinsumo>'` evita compensar dos veces si la factura
  se re-valida, y permite filtrar estos movimientos en informes.
- El trigger corre **dentro** de la transacción de la validación: si falla, se revierte todo.
- Resultado neto: "2 × Hamburguesa clásica, Sin cebolla" → sale 2 panes, 2 carnes, 2 lonjas,
  y la cebolla sale y vuelve a entrar (neto 0) con su motivo trazable.

> **Por qué compensar y no reemplazar el descargo nativo:** el núcleo ya descarga kits de forma
> correcta y probada (incluye sub-kits y el almacén del terminal). Compensar solo lo omitido es
> menos código, no pelea con futuras versiones de TakePOS y deja un rastro explícito de cada "Sin…".

### 11.3 Trigger `BILL_UNVALIDATE`

Si una factura vuelve a borrador indicando almacén, el núcleo revierte sus movimientos. El módulo revierte sus
compensaciones (movimientos con `inventorycode LIKE 'FFP-<id>-%'`) con el movimiento contrario.

### 11.4 Stock insuficiente

- Si `STOCK_DISALLOW_NEGATIVE_TRANSFER` está activo y falta un insumo, la validación falla en el
  cobro con el error nativo. Recomendado en v1: **permitir negativo** y vigilar con el informe de
  stock bajo / alertas (`seuil_stock_alerte` en cada insumo).
- Fase 3: el popup puede mostrar "Agotado" en una tarjeta si algún insumo obligatorio está en 0
  (consulta a `product_stock` del almacén del terminal).

---

## 12. Inventario: carga (compras, ajustes, mermas)

Todo con funciones **nativas** de Dolibarr; el módulo no agrega pantallas para esto.

| Caso | Cómo se hace | Efecto |
|---|---|---|
| **Compra de insumos** | Compras → Pedido a proveedor → Validar → **Recepción** al almacén Cocina (o Depósito) | Entrada de stock y actualización del PMP |
| Compra sin pedido | Factura de proveedor + **Producto → Stock → Corregir stock (entrada)** con precio | Entrada de stock |
| **Traspaso** Depósito → Cocina | Producto → Stock → **Transferir** (o Stocks → Traspaso masivo) | Movimiento entre almacenes |
| **Merma / desperdicio** | Producto → Stock → **Corregir stock (salida)** con etiqueta "Merma: …" y código de inventario `MERMA-AAAAMMDD` | Salida trazable |
| **Consumo interno / comida del personal** | Corregir stock (salida) con etiqueta "Consumo interno" | Salida trazable |
| **Conteo físico** | Stocks → **Inventarios** → nuevo inventario del almacén Cocina → contar → validar | Ajusta diferencias con movimientos |
| **Producción previa** (ej. salsa de la casa por lote) | Opción A: insumo comprado en kg. Opción B (fase 4): módulo **Fabricación (MRP)** con BOM: consume ingredientes y produce "Salsa de la casa (L)" | Stock del preparado |
| **Carga inicial** | Herramientas → **Importar** → "Stocks de productos" (CSV: ref, almacén, cantidad) o Inventario inicial | Stock inicial |

**Recetas (kits):** Producto → pestaña **Kit / Productos virtuales** → agregar cada insumo con su
cantidad y marcar "incdec". Importación masiva de recetas: Herramientas → Importar →
"Productos virtuales / kits" (`product_association`: ref padre, ref hijo, cantidad, incdec).

**Informes útiles (nativos):** Stocks → Movimientos (filtrar por almacén Cocina, por
`inventorycode FFP-%` para ver omisiones), Valor del stock por almacén, Productos con stock bajo.

---

## 13. Devoluciones y anulaciones

| Situación | Flujo | Inventario |
|---|---|---|
| Quitar algo **antes de cobrar** | Popup → Quitar (borra el grupo completo) o borrar la línea en el ticket nativo | Nada que revertir (aún es borrador) |
| Borrar la línea principal desde el **ticket nativo** | Sus adicionales quedan "huérfanos"; `getCart` los muestra sueltos con aviso para quitarlos | — |
| **Devolución** después de cobrar | TakePOS → Historial → ticket → **Nota de crédito** (copia las líneas con sus extrafields) | El núcleo reingresa los insumos de las recetas; el trigger **descuenta** de nuevo lo omitido (sección 11.2) para que el neto sea correcto |
| Volver la factura a borrador | Solo administradores (`Facture::setDraft($user, $idwarehouse)`) | El núcleo revierte el stock si se indica almacén; `BILL_UNVALIDATE` revierte las compensaciones |

---

## 14. Catálogo de ejemplo

Precios y cantidades de ejemplo (iguales al demo). Las recetas son orientativas; cada negocio
ajusta las suyas.

### 14.1 Insumos (rol `insumo`, `tosell = 0`, almacén Cocina)

| Ref | Insumo | Unidad | Categoría |
|---|---|---|---|
| INS-PAN-HAMB | Pan de hamburguesa | unidad | Panadería |
| INS-PAN-PERRO | Pan de perro | unidad | Panadería |
| INS-CARNE-150 | Carne de hamburguesa 150 g | unidad | Proteínas |
| INS-POLLO-CRISPY | Pechuga empanizada | unidad | Proteínas |
| INS-SALCHICHA | Salchicha | unidad | Proteínas |
| INS-TOCINETA | Tocineta | tira | Proteínas |
| INS-HUEVO | Huevo | unidad | Proteínas |
| INS-QUESO-AMA | Queso amarillo | lonja | Lácteos |
| INS-QUESO-RALL | Queso rallado | kg | Lácteos |
| INS-CEBOLLA | Cebolla | kg | Vegetales |
| INS-TOMATE | Tomate | kg | Vegetales |
| INS-LECHUGA | Lechuga | kg | Vegetales |
| INS-REPOLLO | Repollo | kg | Vegetales |
| INS-PAPITAS | Papitas (bolsa) | kg | Desechables |
| INS-SALSAS | Salsas (mayonesa, ketchup, mostaza) | kg | Salsas |
| INS-LATA-COLA | Lata Cola 355 ml | unidad | Bebidas |
| INS-LATA-NAR | Lata Naranja 355 ml | unidad | Bebidas |
| INS-HEL-CHOC | Helado chocolate | L | Helados |
| INS-HEL-VAIN | Helado vainilla | L | Helados |
| INS-CONO | Cono | unidad | Desechables |
| INS-ENVASE | Envase para llevar | unidad | Desechables |

### 14.2 Recetas

| Producto (rol) | Pestaña / categoría | Precio | Receta (insumo × cantidad por unidad) |
|---|---|---|---|
| Hamburguesa clásica (plato) | Hamburguesas | 5,00 | Pan hamb. ×1 · Carne 150 g ×1 · Queso amarillo ×1 · Lechuga ×0,02 · Tomate ×0,03 · Cebolla ×0,03 · Salsas ×0,03 |
| Hamburguesa doble (plato) | Hamburguesas | 7,50 | Pan ×1 · Carne ×2 · Queso ×2 · Tocineta ×2 · Lechuga ×0,02 · Tomate ×0,03 · Cebolla ×0,03 · Salsas ×0,04 |
| Perro clásico (plato) | Perros calientes | 2,00 | Pan perro ×1 · Salchicha ×1 · Repollo ×0,03 · Cebolla ×0,02 · Papitas ×0,01 · Salsas ×0,02 |
| Refresco lata (plato sin receta, `stockable_product = 0`) | Refrescos | 1,50 | — (la receta está en la opción) |
| Cola (opción) | Sabores refresco | 0,00 | Lata Cola ×1 |
| Naranja (opción) | Sabores refresco | 0,00 | Lata Naranja ×1 |
| Agua mineral (reventa, un toque) | Refrescos | 1,00 | — (stock propio) |
| Barquilla doble (plato, opciones min 1 / max 2) | Helados | 3,00 | Cono ×1 |
| Chocolate (opción) | Sabores helado | 0,00 | Helado chocolate ×0,09 |
| Queso amarillo (adicional) | Adicionales hamburguesa | 0,75 | Queso amarillo ×1 |
| Tocineta (adicional) | Adicionales hamburguesa | 1,00 | Tocineta ×2 |
| Papitas extra (adicional) | Adicionales perro | 0,50 | Papitas ×0,01 |

### 14.3 Condiciones

| Pestaña | Condición | Insumo que no se descuenta | Modo |
|---|---|---|---|
| Hamburguesas | Sin cebolla | Cebolla | total |
| Hamburguesas | Sin ensalada | Lechuga | total |
| Hamburguesas | Sin ensalada | Tomate | total |
| Hamburguesas | Sin tomate | Tomate | total |
| Hamburguesas | Sin queso | Queso amarillo | total |
| Hamburguesas | Sin salsas | Salsas | total |
| Hamburguesas | Salsas aparte | — (solo texto) | — |
| Hamburguesas | Carne bien cocida | — | — |
| Perros calientes | Sin cebolla | Cebolla | total |
| Perros calientes | Sin repollo | Repollo | total |
| Perros calientes | Sin papitas | Papitas | total |
| Perros calientes | Sin salsas | Salsas | total |
| Helados | En vaso | — | — (fase 4: cambiar cono por vaso) |

---

## 15. Permisos, idiomas y seguridad

### 15.1 Permisos

| Id | Código | Etiqueta | Quién |
|---|---|---|---|
| 75020011 | `fastfoodpos->use` | Usar PedidosFood en el TakePOS | Cajeros |
| 75020021 | `fastfoodpos->config` | Configurar pestañas y condiciones | Encargado |

Comprobación: `$user->hasRight('fastfoodpos', 'use')`. El botón no aparece si el usuario no lo tiene.

### 15.2 Idiomas (`langs/es_ES/fastfoodpos.lang`)

```
ModuleFastFoodPOSName = FastFood POS
ModuleFastFoodPOSDesc = Botón PedidosFood en TakePOS: toma de pedidos de comida rápida con adicionales, condiciones y descargo de insumos por receta
PedidosFood = PedidosFood
FfpPedidoActual = Pedido actual
FfpFinalizar = Finalizar pedido
FfpAgregar = Agregar al pedido
FfpGuardarCambios = Guardar cambios
FfpCondiciones = Condiciones especiales
FfpCondicionesAyuda = no se cobran · van a cocina
FfpAdicionales = Adicionales
FfpNota = Nota para cocina
FfpUnToque = 1 toque
FfpComerAqui = Comer aquí
FfpParaLlevar = Para llevar
FfpOmitido = Omitido: %s – Factura %s
FfpErrorOpcionObligatoria = Elige %s antes de agregar
FfpErrorMaxAdicionales = Máximo %s adicionales por producto
FfpErrorFacturaNoBorrador = La venta ya fue cobrada; abre una nueva
```

### 15.3 Seguridad

- Endpoints: `top_httphead('application/json')`, permisos en **todas** las acciones, `POST` + token para escribir (no definir `NOCSRFCHECK`).
- `GETPOST(..., 'none')` solo para JSON que luego se decodifica y valida campo por campo; enteros con `GETPOSTINT`.
- SQL siempre con `(int)` y `$db->escape()`; `MAIN_DB_PREFIX` / `$db->prefix()`; filtrar `entity` con `getEntity('product')`.
- Escapar todo texto en el JS (`esc()`), como en el demo.

---

## 16. README del módulo (manual de configuración y uso)

El módulo **debe** incluir `custom/fastfoodpos/README.md`: un manual paso a paso para quien
instala y configura el sistema (implementador o encargado) y para quien lo usa a diario
(cajero). Es un entregable obligatorio, no opcional: sin él el módulo no se considera terminado.

### 16.1 Reglas del README

- **Idioma:** español, tono directo, en segunda persona ("Entra a…", "Marca…").
- **Ruta de menú exacta** de Dolibarr 24 en cada paso (**Inicio → Configuración → Módulos**…), en negrita.
- **Un paso = una acción** con su resultado esperado ("Debes ver…"). Numerado.
- **Orden obligatorio:** la configuración se hace en el orden de la sección 16.3 (cada paso depende del anterior: no se puede armar una receta sin insumos ni asignar condiciones sin categorías).
- **Sin IDs fijos:** se nombran categorías y productos, nunca un `rowid`.
- **Capturas** *(recomendado)* en `img/readme/` con nombre `NN-descripcion.png` y enlazadas en el paso.
- **Se actualiza con cada versión:** si cambia un ajuste, un menú o un extrafield, el README cambia en el mismo commit. La versión del README = versión del descriptor (`$this->version`).
- Debe coincidir con este `fastfood.md`; si hay contradicción, se corrige el README.

### 16.2 Índice obligatorio del README

Introducción (qué hace el módulo, 1 párrafo + imagen del popup), y luego:

1. Requisitos
2. Instalación
3. Configuración inicial de Dolibarr y TakePOS
4. Almacén e insumos
5. Categorías
6. Productos y recetas
7. Adicionales y opciones
8. Condiciones especiales
9. Configuración de FastFood POS (pestañas y terminales)
10. Permisos de los cajeros
11. Carga inicial de inventario
12. Prueba de control (antes de abrir al público)
13. Uso diario en el POS (cajero)
14. Inventario en el día a día (compras, traspasos, mermas, conteos)
15. Devoluciones
16. Solución de problemas
17. Preguntas frecuentes
18. Copiar la configuración a otro local (exportar / importar)
19. Actualizar o desinstalar el módulo

### 16.3 Contenido mínimo de cada sección (texto base)

El implementador parte de este texto y lo ajusta a lo que realmente construyó.

~~~markdown
# FastFood POS — Manual de configuración y uso

Versión del módulo: 1.0 · Dolibarr 24.0.x · TakePOS

FastFood POS agrega al TakePOS el botón **PedidosFood**. Con él tomas pedidos de comida
rápida en una pantalla táctil (hamburguesas, perros, refrescos, postres, helados), con
**adicionales** que se cobran, **condiciones** como "Sin cebolla" que no se cobran, y
**opciones** como el sabor del refresco. El pedido queda en el ticket del TakePOS para cobrar e
imprimir como siempre. Al cobrar, Dolibarr descuenta del inventario los **insumos** de cada
receta (pan, carne, queso…), excepto los que el cliente pidió quitar.

## 1. Requisitos

- Dolibarr **24.0** o superior, PHP 8.1+.
- Usuario **administrador** para configurar.
- Módulos: Productos, Categorías, Facturas, Stocks, TakePOS. Recomendados: Pedidos a proveedor, Recepciones, Inventarios.
- El módulo **Lotes/Series** debe estar **desactivado**.

## 2. Instalación

1. Copia la carpeta `fastfoodpos` dentro de `htdocs/custom/` (debe quedar `htdocs/custom/fastfoodpos/core/modules/modFastFoodPOS.class.php`).
2. Entra a **Inicio → Configuración → Módulos**, busca **FastFood POS** y actívalo.
   Debes ver el módulo en verde. Al activarlo se crean los campos adicionales, la tabla de condiciones y el permiso.
3. Si Dolibarr pide activar módulos dependientes (TakePOS, Stocks), acepta.
4. Abre **FastFood POS → Configurar → Diagnóstico**. Es normal ver 🔴 en esta etapa: te dice qué falta.
   Sigue los pasos 3 a 10 de este manual y vuelve a correrlo al final.

> **¿Ya tienes el catálogo de otro local o quieres el de ejemplo?** Después del paso 3, ve a
> **FastFood POS → Configurar → Catálogo → Importar** y carga el archivo (o `data/catalogo-ejemplo.json`)
> en modo *Simular* y luego *Solo crear lo nuevo*. Eso reemplaza los pasos 4 a 9; solo revisa precios y recetas.

## 3. Configuración inicial de Dolibarr y TakePOS

1. **Inicio → Configuración → Módulos → Productos (configurar)**: activa **Kits / productos virtuales** = Sí.
2. **Inicio → Configuración → Módulos → Stocks (configurar)**: decide si permites stock negativo. Recomendado al comenzar: permitirlo, para que una venta no se bloquee por un insumo mal contado.
3. **Inicio → Configuración → Otras configuraciones**: confirma que **no** existan (o estén en 0)
   `PRODUIT_SOUSPRODUITS_ALSO_ENABLE_PARENT_STOCK_MOVE` y `PRODUIT_SOUSPRODUITS_ALWAYS_DISABLE_CHILDREN_STOCK_MOVE`.
4. **TakePOS → Configuración → General**: **Agrupar el mismo producto** = **No**.
5. **TakePOS → Configuración → Terminal 1** (repite para cada terminal):
   - Cliente por defecto: "Cliente contado" (créalo si no existe).
   - Almacén: **Cocina** (se crea en el paso 4.1).
   - No descontar stock: **No**.
   - Formas de pago: las que uses (efectivo USD, efectivo Bs, pago móvil, punto).

## 4. Almacén e insumos

1. **Productos → Almacenes → Nuevo almacén**: crea **Cocina** (y **Depósito** si guardas compras aparte).
2. Crea cada insumo en **Productos → Nuevo producto**:
   - Ref: `INS-…` (ej. `INS-CEBOLLA`).
   - **En venta: No** · En compra: Sí.
   - Unidad: la que compras y cuentas (unidad, kg, L).
   - Campo **Rol en PedidosFood**: *Insumo*.
   - Stock límite para alerta: el mínimo que quieres tener.
3. Asigna cada insumo a su categoría de **FFP · Insumos** (Panadería, Proteínas, Vegetales…).

> Para muchos insumos usa **Herramientas → Importar** con un CSV de productos.

## 5. Categorías

En **Productos → Categorías → Nueva categoría** crea este árbol (los nombres pueden cambiar, la estructura no):

    PedidosFood            ← raíz del POS
    ├── Hamburguesas
    ├── Perros calientes
    ├── Refrescos
    ├── Postres
    └── Helados
    FFP · Adicionales      ← NO va dentro de PedidosFood
    ├── Adicionales hamburguesa
    ├── Adicionales perro
    ├── Adicionales helado
    └── Adicionales postre
    FFP · Opciones
    ├── Sabores refresco
    ├── Sabores helado
    └── Frutas jugo
    FFP · Insumos
    └── (Panadería, Proteínas, Vegetales, Salsas, Lácteos, Bebidas, Helados, Desechables)

Luego, en **TakePOS → Configuración → General**, elige **PedidosFood** como categoría raíz.
Debes ver en el POS solo las 5 pestañas, nunca los insumos.

> Opcional: dale un **color** a cada pestaña en la ficha de la categoría; el popup lo usa.

## 6. Productos y recetas

### 6.1 Platos (hamburguesas, perros, postres caseros, helados)

1. **Productos → Nuevo producto**: ref, nombre, **precio de venta**, IVA, **En venta: Sí**.
2. **Rol en PedidosFood**: *Plato*.
3. Asígnalo a su pestaña (ej. Hamburguesas).
4. Pestaña **Kit / Productos virtuales** de la ficha → agrega cada insumo con la cantidad **por unidad vendida** y marca **Incrementar/disminuir stock** (incdec) en todos.
   Ejemplo Hamburguesa clásica: Pan ×1 · Carne 150 g ×1 · Queso ×1 · Lechuga ×0,02 · Tomate ×0,03 · Cebolla ×0,03 · Salsas ×0,03.
5. Revisa que el plato **no** tenga stock propio: su stock lo ponen sus insumos.

### 6.2 Productos de reventa (agua, malta, postres empacados)

1. Producto con precio, **En venta: Sí**, **Rol**: *Reventa*, sin kit.
2. Si quieres que se agregue con un toque: **Al tocarlo en el POS** = *Un toque*.

### 6.3 Productos con sabor a elegir (refresco lata, jugos, helados)

1. Crea el plato (ej. "Refresco lata", precio 1,50) **sin kit** y con **Gestión de stocks = No**.
2. Llena en su ficha:
   - **Categoría de opciones**: Sabores refresco.
   - **Título de las opciones**: Sabor · **Mínimo**: 1 · **Máximo**: 1.
3. Las recetas van en cada opción (sección 7.2).

## 7. Adicionales y opciones

### 7.1 Adicionales (se cobran)

1. Producto "Queso amarillo (adicional)", precio 0,75, **En venta: Sí**, **Rol**: *Adicional*.
2. Categoría: **Adicionales hamburguesa**.
3. Kit: Queso amarillo (insumo) ×1, con incdec.

### 7.2 Opciones (sabores, frutas)

1. Producto "Cola", precio 0,00 (o recargo), **En venta: Sí**, **Rol**: *Opción*.
2. Categoría: **Sabores refresco**.
3. Kit: Lata Cola 355 ml ×1, con incdec. Para helado: Helado chocolate (L) ×0,09.

> Adicionales y opciones **no** deben estar dentro de la categoría raíz PedidosFood.

## 8. Condiciones especiales

**Inicio → Configuración → Módulos → FastFood POS → pestaña Condiciones**

1. Elige la pestaña (ej. Hamburguesas) → **Nueva condición**.
2. Escribe el texto tal como lo leerá cocina: "Sin cebolla".
3. **Insumo que no se descuenta**: Cebolla. Modo: **Total**.
4. Para condiciones de solo texto ("Salsas aparte", "Bien cocida") deja el insumo vacío.
5. Si una condición quita varios insumos ("Sin ensalada" = lechuga + tomate), crea una fila por insumo con **el mismo texto**: el POS mostrará un solo botón.
6. Usa **Copiar condiciones de otra categoría** para no repetir el trabajo.

## 9. Configuración de FastFood POS

**Inicio → Configuración → Módulos → FastFood POS → Configurar**

Pestaña **General**:
1. Texto y color del botón (por defecto **PedidosFood**, rojo).
2. Para cada pestaña del POS:
   - **Al tocar un producto**: *Abrir ventana* (Hamburguesas, Perros, Postres, Helados) o *Un toque* (Refrescos).
   - **Categoría de adicionales**: Hamburguesas → Adicionales hamburguesa; Perros → Adicionales perro…
   - **Máx. adicionales** (0 = sin límite).
3. Tasa BCV: desde Multimoneda o manual.

Pestaña **Terminal N**: categorías a ocultar en ese terminal (opcional).

> Lo que pongas en la ficha de un producto manda sobre la configuración de su pestaña.

## 10. Permisos de los cajeros

**Usuarios y grupos → Grupo "Cajeros" → Permisos**:
- TakePOS → Usar el punto de venta.
- FastFood POS → **Usar PedidosFood**.
- (Encargado) FastFood POS → Configurar pestañas y condiciones.

## 11. Carga inicial de inventario

Elige una opción:
- **Inventario inicial**: **Productos → Inventarios → Nuevo** → almacén Cocina → cuenta cada insumo → **Validar**.
- **Importación**: **Herramientas → Importar → Stock de productos** con CSV (ref, almacén, cantidad).
- **Corrección manual**: ficha del insumo → **Stock → Corregir stock** (entrada).

## 12. Prueba de control (antes de abrir al público)

0. Corre **FastFood POS → Configurar → Diagnóstico**: no debe haber ningún 🔴.
1. Anota el stock de Pan, Carne y Cebolla.
2. En el POS: **PedidosFood → Hamburguesas → Hamburguesa clásica → Sin cebolla → + Queso → cantidad 2 → Agregar → Finalizar pedido**.
3. El ticket debe mostrar 2 Hamburguesa clásica ("Sin cebolla | Con: Queso amarillo") y 2 Queso amarillo (adicional).
4. Cobra en efectivo.
5. Revisa **Productos → Stocks → Movimientos**: Pan −2, Carne −2, Queso −4 (2 de receta + 2 adicionales), Cebolla −0,06 y **+0,06** ("Omitido: Sin cebolla").
6. Si algo no cuadra, ve a **Solución de problemas** antes de vender.

## 13. Uso diario en el POS (cajero)

1. Abre **TakePOS** y elige la mesa o "Para llevar".
2. Toca **PedidosFood** (botón rojo en la barra de acciones).
3. Toca una **pestaña** y luego un **producto**:
   - Productos con insignia **1 toque** se agregan de inmediato.
   - Los demás abren su ventana:
     - **Opción obligatoria** (sabor): elige antes de agregar.
     - **Condiciones especiales**: toca para marcar en rojo ("✕ Sin cebolla"). No se cobran.
     - **Adicionales**: + / − por cada unidad del producto. Se cobran.
     - **Nota para cocina**: texto libre corto.
     - **Cantidad** y **Agregar al pedido**.
4. A la derecha ves el **Pedido actual**: puedes **Editar**, **Quitar** o **Vaciar**.
5. Toca **Finalizar pedido**: el popup se cierra y el ticket del POS muestra todo.
6. Cobra con **Pagar** como siempre e imprime el ticket.

Reglas para el cajero:
- No agregues el mismo plato desde la rejilla normal del POS si lleva condiciones o adicionales: usa siempre PedidosFood.
- Si borras una hamburguesa desde el ticket normal, borra también sus adicionales (o hazlo desde PedidosFood con **Quitar**, que borra todo junto).

## 14. Inventario en el día a día

| Necesito… | Hago… |
|---|---|
| Registrar una compra | **Compras → Pedido a proveedor** → validar → **Recepción** en Cocina |
| Pasar mercancía del depósito a cocina | Ficha del insumo → **Stock → Transferir** |
| Registrar una merma (se dañó, se cayó) | Ficha del insumo → **Corregir stock (salida)**, etiqueta "Merma: motivo" |
| Comida del personal | **Corregir stock (salida)**, etiqueta "Consumo interno" |
| Conteo semanal | **Productos → Inventarios → Nuevo** → contar → validar |
| Ver qué se omitió por condiciones | **Stocks → Movimientos**, filtro código de inventario `FFP-` |
| Ver qué falta comprar | **Productos → Stocks → Productos con stock bajo la alerta** |

## 15. Devoluciones

1. **TakePOS → Historial** → abre el ticket → **Nota de crédito**.
2. Dolibarr reingresa los insumos de las recetas y el módulo corrige lo omitido. No hagas ajustes manuales.

## 16. Solución de problemas

**Primero corre el Diagnóstico** (FastFood POS → Configurar → Diagnóstico): detecta la mayoría de estos casos y te lleva a la pantalla donde se corrigen.

| Síntoma | Causa probable | Solución |
|---|---|---|
| No aparece el botón PedidosFood | Falta permiso o la constante de hooks | Revisa permisos; desactiva y reactiva el módulo |
| El popup sale vacío | Raíz del TakePOS mal elegida | TakePOS → General → categoría raíz = PedidosFood |
| No aparecen adicionales | Pestaña sin categoría de adicionales, o producto con "Sin adicionales" | Configurar → General; revisa la ficha del producto |
| No se descuentan insumos | Kits desactivados, incdec sin marcar, terminal con "No descontar stock" o sin almacén | Pasos 3.1, 6.1.4 y 3.5 |
| Se descuenta la cebolla aunque se pidió "Sin cebolla" | La condición no tiene insumo o el insumo no está en la receta | Pestaña Condiciones; revisa el kit del plato |
| Error de stock al cobrar | Stock negativo prohibido y falta un insumo | Carga el insumo o permite stock negativo |
| Una hamburguesa "Sin cebolla" se sumó a una normal | "Agrupar el mismo producto" activo | TakePOS → General → No |
| El popup muestra una versión vieja | Caché del navegador | Ctrl+F5 |

## 17. Preguntas frecuentes

- **¿Puedo cambiar una receta?** Sí, en el kit del plato. Las ventas anteriores no cambian.
- **¿Puedo usar el POS sin PedidosFood?** Sí, la rejilla nativa sigue funcionando; solo pierdes condiciones y adicionales.
- **¿Funciona con varias cajas?** Sí, cada terminal usa su almacén y su configuración.
- **¿Qué pasa si cierro el popup sin finalizar?** Lo agregado ya está en el ticket; nada se pierde.

## 18. Copiar la configuración a otro local

1. En el local de origen: **Configurar → Catálogo → Exportar** (decide si incluyes precios).
2. En el local nuevo: instala el módulo y haz los pasos 2 y 3 de este manual.
3. **Configurar → Catálogo → Importar** → *Simular* → revisa el informe → *Solo crear lo nuevo*
   (o *Crear y actualizar* si ya tenía catálogo). Marca **no tocar precios** si la sucursal tiene los suyos.
4. Configura almacén y terminal (paso 3.5), permisos (paso 10) y carga el stock inicial (paso 11).
5. Corre el **Diagnóstico** y la **prueba de control** (paso 12).

## 19. Actualizar o desinstalar

- **Antes de actualizar:** exporta el catálogo (**Configurar → Catálogo → Exportar**) como respaldo.
- **Actualizar el módulo:** reemplaza la carpeta `custom/fastfoodpos`, luego desactiva y reactiva el módulo. Tus productos, recetas y condiciones se conservan.
- **Después de actualizar Dolibarr:** corre el **Diagnóstico** antes de abrir la caja; el grupo *Compatibilidad con TakePOS* avisa si la actualización cambió algo que el módulo usa.
- **Desinstalar:** desactiva el módulo. Los campos adicionales y la tabla de condiciones quedan en la base de datos (no se borran datos de ventas ni de stock).
~~~

### 16.4 Criterio de aceptación del README

Una persona que **no** participó en el desarrollo instala el módulo en un Dolibarr 24 limpio
siguiendo solo el README, carga el catálogo de ejemplo (sección 14) y la **prueba de control**
(paso 12 del README) da los movimientos esperados. Si necesitó preguntar algo, el README se
corrige.

---

## 17. Plan de construcción por fases

| Fase | Entregable | Criterio de "listo" |
|---|---|---|
| **1. Esqueleto** | Descriptor, extrafields, tabla de condiciones, permisos, Setup general, botón PedidosFood que abre un popup vacío, **Diagnóstico grupos A–D** | El botón aparece solo con permiso; activar/desactivar el módulo no da errores |
| **2. Toma de pedido** | `api.php` (`getData`, `getProduct`, `addItem`, `getCart`, `removeItem`, `clearCart`) + `picker.js` portado del demo | Se arma un pedido completo y el ticket nativo lo muestra tras `Refresh()`; cobro nativo OK |
| **3. Inventario** | Trigger `BILL_VALIDATE` / `BILL_UNVALIDATE` con compensación; `admin/condiciones.php`; **Diagnóstico grupo E** | Pruebas 18.2 en verde |
| **4. Pulido** | `updateItem` (Editar), tipo de pedido, número de pedido del día, tasa BCV, "Agotado" | Pruebas 18.3 en verde |
| **4a. Portabilidad** | Exportar / importar catálogo (8.5) y `data/catalogo-ejemplo.json` | Pruebas 18.4 en verde |
| **4b. Manual** | `README.md` completo según la sección 16 (se empieza en la fase 1 y se completa aquí) | Criterio 16.4 cumplido |
| **5. Opcional** | Combos (plato + bebida + papas a precio especial), cambio de insumo ("En vaso" = vaso en vez de cono), comanda separada para cocina, MRP para preparados | — |

---

## 18. Pruebas de aceptación

### 18.1 Pedido y factura

1. Usuario sin permiso `fastfoodpos->use` → el botón **no** aparece.
2. Abrir PedidosFood en una mesa nueva (sin factura) → al agregar un plato se crea `(PROV-POS1-<mesa>)`.
3. Agua mineral (un toque) → se agrega sin ventana.
4. Refresco lata sin elegir sabor → error "Elige sabor"; con Cola → 2 líneas (Refresco lata 1,50 + Cola 0,00).
5. Perro clásico con 4 papitas extra y máximo 3 → el servidor responde 400.
6. 2 × Hamburguesa clásica + Queso + Tocineta → total 13,50 (5,00 + 0,75 + 1,00) × 2.
7. Quitar el ítem desde el popup → desaparecen sus 3 líneas.
8. Finalizar → el popup se cierra y el ticket nativo se actualiza sin recargar la página.
9. Agregar un producto desde la rejilla nativa con un ítem personalizado igual ya en el ticket → **no** se mezclan (requiere `TAKEPOS_GROUP_SAME_PRODUCT` = No).

### 18.2 Inventario

1. Stock inicial: Cebolla 1,000 kg; Pan hamb. 10; Carne 10.
2. Vender y cobrar 2 × Hamburguesa clásica **Sin cebolla**.
3. Esperado: Pan 8, Carne 8, Cebolla **1,000** (sale 0,060 y entra 0,060 con etiqueta "Omitido: Sin cebolla").
4. El stock de "Hamburguesa clásica" no cambia (padre del kit).
5. Nota de crédito de esa venta → Pan 10, Carne 10, Cebolla 1,000.
6. Re-validar una factura no duplica la compensación (`inventorycode` único).
7. Terminal con `CASHDESK_NO_DECREASE_STOCK = 1` → no hay movimientos nativos ni compensaciones.
8. Compra por recepción de 5 kg de cebolla → Cebolla 6,000 y PMP actualizado.

### 18.3 Interfaz

1. Tablet 1280×800 y teléfono 390 px: sin scroll horizontal; botones táctiles ≥ 44 px.
2. Editar un ítem conserva opciones, condiciones, adicionales y nota.
3. El equivalente en Bs usa la tasa configurada.

### 18.4 Portabilidad

1. Instalación limpia de Dolibarr 24 → activar el módulo → **Diagnóstico** marca en 🔴 kits, terminal y raíz; tras corregirlos según los enlaces, todo 🟢 salvo el grupo E (catálogo vacío).
2. Activar `TAKEPOS_GROUP_SAME_PRODUCT` → Diagnóstico 🔴 `GROUP_SAME`; aviso visible en Setup y franja roja en el popup solo para administradores.
3. Renombrar en una copia de pruebas `function Refresh(` en `takepos/index.php` → 🔴 `TP_JS_REFRESH` (prueba del grupo D; restaurar después).
4. Importar `data/catalogo-ejemplo.json` en **Simular** → informe sin escribir nada; en **Solo crear lo nuevo** → catálogo creado y Diagnóstico 🟢.
5. Exportar desde la instalación A e importar en B (IDs distintos) → mismas pestañas, recetas, condiciones y configuración por pestaña; la prueba de control del README da igual en ambas.
6. Importar un archivo con una receta que apunta a una ref inexistente → error, **nada** se crea (todo o nada).
7. Reimportar el mismo archivo en **Crear y actualizar** → 0 creados, recetas reemplazadas sin duplicar componentes.

---

## 19. Lecciones de SabrooskiPOS que aplican aquí

1. **Rutas:** a `top_htmlhead()` se le pasan rutas relativas (`/fastfoodpos/css/picker.css`);
   `dol_buildpath()` antepone `DOL_URL_ROOT` y `/custom`. Pasar `DOL_URL_ROOT` delante genera doble prefijo (404).
2. **Hook correcto:** botones de TakePOS = `ActionButtons` en contexto `takeposfrontend`; cada
   resultado es un **array de botones**; devolver `0` para conservar los nativos.
3. **La constante de hooks** (`MAIN_MODULE_FASTFOODPOS_HOOKS`) se crea al **activar** el módulo.
   Si el botón no aparece tras copiar archivos: desactivar y volver a activar.
4. **`window.FASTFOODPOS` antes de `picker.js`**, que se carga al final del `<body>`.
5. **Productos base desde el endpoint nativo** `getProducts` (no duplicar la consulta del POS).
6. **`invoiceid` puede ser 0** en una venta nueva: resolver siempre la factura por la ref `(PROV-POS<term>-<place>)` en el servidor.
7. **Fuente de verdad = factura provisional**: el pedido no se guarda en memoria del navegador;
   si el popup se cierra, el pedido sigue en el ticket.
8. **Agregar líneas en el servidor y en una transacción** (no encadenar `addline` + `addnote`
   desde el navegador como en la primera versión de Sabrooski).
