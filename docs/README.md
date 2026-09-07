# Subconsultas y Procedimientos almacenados

Devuelve una fila por item

```sql
SELECT
    s.id AS sale_id, s.sale_number, s.status, s.total_amount, s.created_at,
    c.name AS customer_name, c.document_number AS customer_doc,
    u.name AS vendedor,
    det.*
FROM sales s
LEFT JOIN customers c ON c.id = s.customer_id
LEFT JOIN users u ON u.id = s.user_id
LEFT JOIN LATERAL (
    SELECT COALESCE(SUM(sp.amount),0) AS total_pagado,
           COUNT(*) AS num_pagos,
           jsonb_agg(jsonb_build_object('metodo', pm2.name, 'monto', sp.amount, 'ref', sp.reference) ORDER BY sp.created_at) FILTER (WHERE sp.id IS NOT NULL) AS pagos_json
    FROM sale_payments sp
    LEFT JOIN payment_methods pm2 ON pm2.id = sp.payment_method_id
    WHERE sp.sale_id = s.id
) pay ON true
LEFT JOIN LATERAL (
    SELECT si.product_name, si.product_barcode, si.quantity, si.unit_price, si.discount_percentage, si.tax_rate, si.subtotal, si.tax_amount, si.total,
           p.barcode AS product_catalog_barcode, p.name AS product_catalog_name
    FROM sale_items si
    LEFT JOIN products p ON p.id = si.product_id
    WHERE si.sale_id = s.id
) det ON true
WHERE s.id = '9578da5b-52bc-4631-91bf-25a5d21368af';
```

Respuesta:

```sql
[{"sale_id":"9578da5b-52bc-4631-91bf-25a5d21368af","sale_number":"FAC0001819","status":"completed","total_amount":"23400.00","created_at":"2026-03-21 16:10:25","customer_name":"Consumidor Final","customer_doc":"222222222222","vendedor":"[Eliminado] crisortega807","product_name":"Viguetas O Principales PVC","product_barcode":"124578","quantity":"6.0000","unit_price":"3900.00","discount_percentage":"0.00","tax_rate":"0.1900","subtotal":"2296.37","tax_amount":"21103.63","total":"23400.00","product_catalog_barcode":"124578","product_catalog_name":"Viguetas O Principales PVC"}]
```

### Procedimiento Almacenado

```sql
CREATE OR REPLACE PROCEDURE sp_venta_detalle_tx(
    IN p_sale_id UUID,
    INOUT p_detalle JSONB DEFAULT NULL,
    INOUT p_codigo TEXT DEFAULT NULL,
    INOUT p_mensaje TEXT DEFAULT NULL
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_venta_existe BOOLEAN := FALSE;
    v_error_code TEXT;
    v_error_msg TEXT;
BEGIN
    IF p_sale_id IS NULL THEN
        p_codigo := 'BAD_REQUEST';
        p_mensaje := 'p_sale_id no puede ser NULL';
        p_detalle := NULL;
        COMMIT;
        RETURN;
    END IF;
    -- SUBTRANSACTION 1: verificar existencia
    BEGIN
        SELECT EXISTS(SELECT 1 FROM sales WHERE id = p_sale_id)
        INTO v_venta_existe;

        IF NOT v_venta_existe THEN
            RAISE EXCEPTION 'Venta no encontrada: %', p_sale_id
                USING ERRCODE = 'P0002';
        END IF;
    EXCEPTION
        WHEN OTHERS THEN
            p_detalle := NULL;
            IF SQLSTATE = 'P0002' THEN
                p_codigo := 'NOT_FOUND';
                p_mensaje := SQLERRM;
            ELSE
                p_codigo := SQLSTATE;
                p_mensaje := format('Error %s: %s', SQLSTATE, SQLERRM);
            END IF;
            COMMIT;
            RETURN;
    END;
    -- Subcommit implícito de la subtransacción 1
    -- SUBTRANSACTION 2: construir expediente completo
    BEGIN
        SELECT jsonb_build_object(
            'cabecera', to_jsonb(s),
            'customer', COALESCE(to_jsonb(c), '{}'::jsonb),
            'vendedor', jsonb_build_object(
                'id', u.id, 'name', u.name,
                'email', u.email, 'role', u.role
            ),
            'payment_method', COALESCE(to_jsonb(pm), '{}'::jsonb),
            'cash_shift', COALESCE(to_jsonb(cs), '{}'::jsonb),
            'branch_id', s.branch_id,

            'items', COALESCE((
                SELECT jsonb_agg(
                    jsonb_build_object(
                        'id', si.id,
                        'product_id', si.product_id,
                        'product_name', si.product_name,
                        'product_barcode', si.product_barcode,
                        'quantity', si.quantity,
                        'unit_price', si.unit_price,
                        'discount_percentage', si.discount_percentage,
                        'tax_rate', si.tax_rate,
                        'subtotal', si.subtotal,
                        'tax_amount', si.tax_amount,
                        'total', si.total
                    ) ORDER BY si.product_name
                ) FROM sale_items si WHERE si.sale_id = s.id
            ), '[]'::jsonb),

            'pagos', COALESCE((
                SELECT jsonb_agg(
                    jsonb_build_object(
                        'id', sp.id,
                        'payment_method_id', sp.payment_method_id,
                        'metodo', pm2.name,
                        'amount', sp.amount,
                        'reference', sp.reference,
                        'bank', sp.bank,
                        'created_at', sp.created_at
                    ) ORDER BY sp.created_at
                ) FROM sale_payments sp
                LEFT JOIN payment_methods pm2 ON pm2.id = sp.payment_method_id
                WHERE sp.sale_id = s.id
            ), '[]'::jsonb),

            'resumen_pagos', jsonb_build_object(
                'total_pagado', COALESCE(
                    (SELECT SUM(amount) FROM sale_payments WHERE sale_id = s.id), 0
                ),
                'num_pagos', (
                    SELECT COUNT(*) FROM sale_payments WHERE sale_id = s.id
                ),
                'saldo_pendiente', s.balance_due
            ),

            'credito', (
                SELECT to_jsonb(cr) FROM credits cr WHERE cr.sale_id = s.id
            ),

            'cuotas_credito', COALESCE((
                SELECT jsonb_agg(to_jsonb(ci) ORDER BY ci.due_date)
                FROM credit_installments ci
                WHERE ci.credit_id = (
                    SELECT id FROM credits WHERE sale_id = s.id
                )
            ), '[]'::jsonb),

            'devoluciones', COALESCE((
                SELECT jsonb_agg(jsonb_build_object(
                    'return', to_jsonb(r),
                    'items', (
                        SELECT COALESCE(jsonb_agg(to_jsonb(ri)), '[]'::jsonb)
                        FROM return_items ri WHERE ri.return_id = r.id
                    )
                ) ORDER BY r.created_at)
                FROM returns r WHERE r.sale_id = s.id
            ), '[]'::jsonb),

            'documentos', COALESCE((
                SELECT jsonb_agg(to_jsonb(sd) ORDER BY sd.created_at)
                FROM sale_documents sd WHERE sd.sale_id = s.id
            ), '[]'::jsonb),
            'timeline', COALESCE((
                SELECT jsonb_agg(to_jsonb(st) ORDER BY st.occurred_at DESC)
                FROM sale_timeline_events st WHERE st.sale_id = s.id
            ), '[]'::jsonb),
            'factura_electronica', (
                SELECT to_jsonb(ei)
                FROM electronic_invoices ei WHERE ei.sale_id = s.id
            ),
            'despachos', COALESCE((
                SELECT jsonb_agg(to_jsonb(d) ORDER BY d.created_at)
                FROM delivery_orders d WHERE d.sale_id = s.id
            ), '[]'::jsonb),
            'adjuntos', COALESCE((
                SELECT jsonb_agg(to_jsonb(sa) ORDER BY sa.created_at)
                FROM sale_attachments sa WHERE sa.sale_id = s.id
            ), '[]'::jsonb),
            'comentarios', COALESCE((
                SELECT jsonb_agg(to_jsonb(sc) ORDER BY sc.created_at)
                FROM sale_comments sc WHERE sc.sale_id = s.id
            ), '[]'::jsonb),

            'totales_verificacion', jsonb_build_object(
                'sum_items_total', COALESCE(
                    (SELECT SUM(total) FROM sale_items WHERE sale_id = s.id), 0
                ),
                'sum_items_tax', COALESCE(
                    (SELECT SUM(tax_amount) FROM sale_items WHERE sale_id = s.id), 0
                )
            )
        )
        INTO p_detalle
        FROM sales s
        LEFT JOIN customers c ON c.id = s.customer_id
        LEFT JOIN users u ON u.id = s.user_id
        LEFT JOIN payment_methods pm ON pm.id = s.payment_method_id
        LEFT JOIN cash_shifts cs ON cs.id = s.shift_id
        WHERE s.id = p_sale_id;
    EXCEPTION
        WHEN OTHERS THEN
            p_detalle := NULL;
            p_codigo := SQLSTATE;
            p_mensaje := format('Error %s: %s', SQLSTATE, SQLERRM);
            COMMIT;
            RETURN;
    END;
    -- Subcommit implícito de la subtransacción 2
    -- ÉXITO: COMMIT al nivel del procedure
    p_codigo := 'OK';
    p_mensaje := 'Expediente generado correctamente';
    COMMIT;
END;
$$;

COMMENT ON PROCEDURE sp_venta_detalle_tx(UUID, JSONB, TEXT, TEXT) IS
'PROCEDIMIENTO TRANSACCIONAL';


```

Llamado a la función

```sql
SELECT jsonb_pretty(sp_get_sale_detail('9578da5b-52bc-4631-91bf-25a5d21368af')) AS expediente;
```

Respuesta:

{% code expandable="true" %}
```sql
{
  "cabecera": {
    "id": "9578da5b-52bc-4631-91bf-25a5d21368af",
    "sale_number": "FAC0001819",
    "status": "completed",
    "invoice_type": "local",
    "subtotal": 2296.37,
    "tax_amount": 21103.63,
    "discount_amount": 0.00,
    "total_amount": 23400.00,
    "balance_due": 0.00,
    "payment_method": "cash",
    "fulfillment_status": "paid",
    "customer_id": "ab4f4375-95af-41c5-8bf4-999da93c235d",
    "customer_name": "Consumidor Final",
    "customer_document_type": "CC",
    "customer_document_number": "222222222222",
    "user_id": "234888af-9454-4c3f-b9b1-00b280bf441c",
    "shift_id": "0df60c64-bc2d-4f41-8504-cc9b2b6c0f10",
    "created_at": "2026-03-21T16:10:25.876932"
  },
  "customer": {
    "id": "ab4f4375-95af-41c5-8bf4-999da93c235d",
    "name": "Consumidor Final",
    "document_type": "CC",
    "document_number": "222222222222",
    "customer_type": "retail",
    "person_type": "natural",
    "department": "Amazonas",
    "municipality": "El Encanto",
    "preferred_payment_method": "cash",
    "tax_responsibility": "No responsable",
    "credit_limit": 0.00,
    "payment_terms": 0,
    "is_active": true
  },
  "vendedor": {
    "id": "234888af-9454-4c3f-b9b1-00b280bf441c",
    "name": "[Eliminado] crisortega807",
    "role": "manager",
    "email": "deleted.234888af-9454-4c3f-b9b1-00b280bf441c@removed.obrablanca"
  },
  "payment_method": {
    "id": "ff510304-f8a9-44ec-ab8a-d152231c2901",
    "name": "Efectivo",
    "payment_type": "Efectivo",
    "requires_reference": false,
    "requires_bank": false,
    "requires_account": false
  },
  "cash_shift": {
    "id": "0df60c64-bc2d-4f41-8504-cc9b2b6c0f10",
    "shift_number": "TUR000086",
    "status": "closed",
    "shift_date": "2026-03-21",
    "started_at": "2026-03-21T08:06:55.995801+00:00",
    "ended_at": "2026-03-21T17:13:06.476+00:00",
    "total_sales_amount": 2500360.00,
    "total_sales_count": 26,
    "cash_sales_amount": 2500360.00,
    "opening_cash_amount": 0,
    "closing_cash_amount": 2412760,
    "expected_cash_amount": 2412760.00,
    "cash_difference": 0
  },
  "items": [
    {
      "id": "d7ea2573-19a2-4d55-badd-22c9ba1f0c6c",
      "product_name": "Viguetas O Principales PVC",
      "product_barcode": "124578",
      "quantity": 6.0000,
      "unit_price": 3900.00,
      "discount_percentage": 0.00,
      "tax_rate": 0.1900,
      "subtotal": 2296.37,
      "tax_amount": 21103.63,
      "total": 23400.00
    }
  ],
  "pagos": [],
  "resumen_pagos": {
    "total_pagado": 0,
    "num_pagos": 0,
    "saldo_pendiente": 0.00
  },
  "credito": null,
  "cuotas_credito": [],
  "devoluciones": [],
  "documentos": [],
  "timeline": [],
  "factura_electronica": null,
  "despachos": [],
  "adjuntos": [],
  "comentarios": [],
  "totales_verificacion": {
    "sum_items_total": 23400.00,
    "sum_items_subtotal": 2296.37,
    "sum_items_tax": 21103.63
  }
}
```
{% endcode %}

