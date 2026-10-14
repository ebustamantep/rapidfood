# RapidFood — Toma de Pedido (demo)

Prototipo táctil de **toma de pedidos para comida rápida**: hamburguesas, perros calientes,
refrescos, postres y helados. Es la referencia visual del futuro botón **"Toma de Pedido"**
del TakePOS de Dolibarr (basado en el prototipo de la heladería Sabrooski).

**Demo en vivo:** https://ebustamantep.github.io/rapidfood/

## Qué muestra

- Pestañas de categorías y rejilla de productos.
- **Condiciones especiales** (no se cobran, van a cocina): sin cebolla, sin ensalada, salsas aparte…
- **Adicionales** con contador y precio (queso, tocineta, huevo, carne extra…), con límite opcional por producto.
- **Opciones obligatorias** (sabor del refresco, sabores del helado).
- Productos de **un toque** (agua, malta) que se agregan sin abrir la ventana.
- Nota libre para cocina, editar/quitar líneas, comer aquí / para llevar.
- Totales en USD y equivalente en Bs con tasa BCV editable.
- **Finalizar pedido** muestra la comanda de cocina.

> Demo estática: los productos y precios son de ejemplo y no procesa ventas reales.

## Personalizar

Todo está en `index.html`. Al inicio del `<script>`:

- `CATEGORIES` — pestañas.
- `PRODUCTS` — productos (`direct: true` = un toque, `choices` = opción obligatoria, `maxExtras` = límite de adicionales).
- `EXTRAS` — adicionales por grupo.
- `COND` — condiciones especiales por grupo.

## Publicar con GitHub Pages

*Settings → Pages → Build and deployment → Source: Deploy from a branch → `main` / `/ (root)` → Save.*

## Módulo Dolibarr

La especificación del módulo **fastfoodpos** (botón *PedidosFood* en TakePOS, categorías,
extrafields, recetas e inventario) está en [`fastfood.md`](fastfood.md).
