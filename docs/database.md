# Database

## Diagramas ER por Dominio

Cada dominio tiene su propio archivo con el diagrama Mermaid optimizado:

| # | Dominio | Archivo | Tablas | Relaciones |
|---|---------|---------|--------|------------|
| 01 | [Catalogo de Productos](dominios/01_catalogo_productos.mmd) | `01_catalogo_productos.mmd` | 16 | 24 |
| 02 | [Corte de Optimizacion](dominios/02_corte_optimizacion.mmd) | `02_corte_optimizacion.mmd` | 4 | 7 |
| 03 | [Proveedores y Compras](dominios/03_proveedores_compras.mmd) | `03_proveedores_compras.mmd` | 4 | 4 |
| 04 | [Inventario](dominios/04_inventario.mmd) | `04_inventario.mmd` | 1 | 0 |
| 05 | [Clientes](dominios/05_clientes.mmd) | `05_clientes.mmd` | 2 | 1 |
| 06 | [Cotizaciones](dominios/06_cotizaciones.mmd) | `06_cotizaciones.mmd` | 2 | 1 |
| 07 | [Ventas](dominios/07_ventas.mmd) | `07_ventas.mmd` | 10 | 8 |
| 08 | [Devoluciones](dominios/08_devoluciones.mmd) | `08_devoluciones.mmd` | 3 | 1 |
| 09 | [Cartera y Credito](dominios/09_cartera_credito.mmd) | `09_cartera_credito.mmd` | 10 | 7 |
| 10 | [Cobranza](dominios/10_cobranza.mmd) | `10_cobranza.mmd` | 4 | 3 |
| 11 | [Caja y Tesoreria](dominios/11_caja_tesoreria.mmd) | `11_caja_tesoreria.mmd` | 10 | 13 |
| 12 | [Contabilidad](dominios/12_contabilidad.mmd) | `12_contabilidad.mmd` | 8 | 7 |
| 13 | [Facturacion Electronica](dominios/13_facturacion_electronica.mmd) | `13_facturacion_electronica.mmd` | 9 | 1 |
| 14 | [Notificaciones y Correo](dominios/14_notificaciones_correo.mmd) | `14_notificaciones_correo.mmd` | 7 | 2 |
| 15 | [WhatsApp](dominios/15_whatsapp.mmd) | `15_whatsapp.mmd` | 5 | 4 |
| 16 | [Tareas Internas](dominios/16_tareas_internas.mmd) | `16_tareas_internas.mmd` | 5 | 4 |
| 17 | [Configuracion y Sucursales](dominios/17_configuracion_sucursales.mmd) | `17_configuracion_sucursales.mmd` | 9 | 3 |
| 18 | [Autenticacion y Seguridad](dominios/18_autenticacion_seguridad.mmd) | `18_autenticacion_seguridad.mmd` | 7 | 5 |
| 19 | [Auditoria y Logs Tecnicos](dominios/19_auditoria_logs.mmd) | `19_auditoria_logs.mmd` | 6 | 0 |
| 20 | [Sistema](dominios/20_sistema.mmd) | `20_sistema.mmd` | 1 | 0 |

## Resumen de Tablas por Dominio

### 01 - Catalogo de Productos (16 tablas)
`categories`, `products`, `product_images`, `product_packagings`, `packaging_presentation_types`, `product_box_specs`, `product_length_specs`, `product_unit_specs`, `product_volume_specs`, `product_weight_specs`, `product_price_history`, `price_lists`, `price_list_items`, `tile_specs`, `pvc_combo_config`, `wood_panels`, `wood_types`

### 02 - Corte de Optimizacion (4 tablas)
`cutting_projects`, `cutting_layouts`, `cutting_pieces`, `optimization_runs`

### 03 - Proveedores y Compras (4 tablas)
`suppliers`, `purchase_orders`, `purchase_items`, `upload_tokens`

### 04 - Inventario (1 tabla)
`stock_movements`

### 05 - Clientes (2 tablas)
`customers`, `customer_credit_settings`

### 06 - Cotizaciones (2 tablas)
`quotes`, `quote_items`

### 07 - Ventas (10 tablas)
`sales`, `sale_items`, `sale_payments`, `sale_attachments`, `sale_comments`, `sale_documents`, `sale_timeline_events`, `pos_sale_idempotency`, `delivery_orders`, `pre_invoices`

### 08 - Devoluciones (3 tablas)
`returns`, `return_items`, `return_command_idempotency`

### 09 - Cartera y Credito (10 tablas)
`credits`, `credit_installments`, `credit_history`, `credit_payments`, `credit_notes`, `customer_advances`, `customer_advance_applications`, `cash_receipts`, `cash_receipt_applications`, `cash_receipt_history`

### 10 - Cobranza (4 tablas)
`collection_cases`, `collection_management_actions`, `collection_payment_promises`, `collection_reminders`

### 11 - Caja y Tesoreria (10 tablas)
`cash_shifts`, `cash_movements`, `shift_command_idempotency`, `bank_accounts`, `bank_reconciliations`, `bank_reconciliation_matches`, `bank_statement_imports`, `bank_statement_lines`, `account_movements`, `reconciliation_items`

### 12 - Contabilidad (8 tablas)
`chart_of_accounts`, `journal_entries`, `journal_lines`, `financial_movements`, `financial_movement_links`, `financial_balances`, `financial_closings`, `financial_close_runs`

### 13 - Facturacion Electronica (9 tablas)
`electronic_invoices`, `adjustment_notes`, `support_documents`, `support_document_lines`, `factus_oauth_tokens`, `factus_provider_credentials`, `dian_numbering_ranges_cache`, `fe_command_idempotency`, `fiscal_email_deliveries`

### 14 - Notificaciones y Correo (7 tablas)
`email_inbox_threads`, `email_inbox_messages`, `email_logs`, `email_outbox`, `email_templates`, `notification_deliveries`, `outbox_events`

### 15 - WhatsApp (5 tablas)
`whatsapp_conversations`, `whatsapp_messages`, `whatsapp_auto_reply_rules`, `whatsapp_auto_reply_history`, `whatsapp_sync_logs`

### 16 - Tareas Internas (5 tablas)
`tasks`, `task_attachments`, `task_collaborators`, `task_comments`, `task_links`

### 17 - Configuracion y Sucursales (9 tablas)
`branches`, `company_settings`, `system_settings`, `features`, `tax_settings`, `taxes`, `measurement_units`, `payment_methods`, `document_numbering`

### 18 - Autenticacion y Seguridad (7 tablas)
`users`, `auth_accounts`, `auth_sessions`, `auth_verifications`, `auth_verification_codes`, `auth_access_logs`, `security_audit_logs`

### 19 - Auditoria y Logs Tecnicos (6 tablas)
`audit_log`, `api_logs`, `app_error_logs`, `db_query_logs`, `system_logs`, `financial_audit_logs`

### 20 - Sistema (1 tabla)
`alembic_version`
