# Trigger

Este Trigger crea una auditoría automática sobre la tabla `cash_movements`. Cada vez que se inserta, actualiza o elimina un registro, ejecuta `audit_trigger_function()` para registrar lo ocurrido.

```sql
CREATE TRIGGER audit_cash_movements
AFTER INSERT OR DELETE OR UPDATE ON cash_movements FOR EACH ROW
EXECUTE FUNCTION audit_trigger_function ()
```

Respuesta:

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

***

Este trigger mantiene sincronizado el resumen de un turno de caja cada vez que cambia la tabla `sales`. Su propósito es recalcular o actualizar los totales del turno afectado.

```sql
CREATE TRIGGER sales_update_shift_summary_trigger
AFTER INSERT OR DELETE OR UPDATE ON sales FOR EACH ROW
EXECUTE FUNCTION trigger_update_shift_summary ()
```

Respuesta

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

***

Este trigger hace prácticamente lo mismo que el anterior, pero sobre la tabla `account_movements`. Su función es mantener actualizado automáticamente el campo `updated_at` cuando un movimiento contable se modifica.

```sql
CREATE TRIGGER account_movements_updated_at
BEFORE UPDATE ON account_movements FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column ()
```

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>
