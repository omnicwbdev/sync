# sync
Sync Firebird to MySQL


## Crontab

```
*/5 * * * * /bin/sh -c 'cd /usr/home/josemario/sync && echo "$(date "+[\%Y-\%m-\%d \%H:\%M:\%S]") Running sync-freebsd" >> sync.log && ./sync-freebsd >> sync.log 2>&1'
```

## Procedures

```sql
CREATE DEFINER = `root` @`%` PROCEDURE `SP_ATUALIZAR_PART_NUMBER` () BEGIN DECLARE rows_affected INT;

UPDATE
  TB_ESTOQUE
SET
  PartNumber = CASE
    WHEN DESCricao LIKE '(%)%' THEN SUBSTRING_INDEX(SUBSTRING_INDEX(DESCricao, ')', 1), '(', -1)
    ELSE NULL
  END;

SET
  rows_affected = ROW_COUNT();

SELECT
  CONCAT(
    'PartNumber atualizado para ',
    rows_affected,
    ' registros'
  ) as Resultado;

END
```

```sql
CREATE DEFINER = `root` @`%` PROCEDURE `UpdateQtdVirtual` () BEGIN
START TRANSACTION;

UPDATE
  TB_ESTOQUE e
  LEFT JOIN (
    SELECT
      r.id_estoque,
      SUM(
        CASE
          WHEN r.status = 'A' THEN r.qtd_reserva
          ELSE 0
        END
      ) AS reservas_ativas
    FROM
      TB_RESERVA r
    GROUP BY
      r.id_estoque
  ) r ON r.id_estoque = e.id_estoque
SET
  e.qtd_virtual = CASE
    -- 1) Se houver qtd_nova > 0, prioridade absoluta
    WHEN COALESCE(e.qtd_nova, 0) > 0 THEN GREATEST(e.qtd_atual - e.qtd_nova, 0)
    -- 2) Se houver reservas ativas (status = 'A'), subtrai do qtd_atual
    WHEN COALESCE(r.reservas_ativas, 0) > 0 THEN GREATEST(e.qtd_atual - r.reservas_ativas, 0)
    -- 3) Caso contrário, mantém qtd_atual
    ELSE e.qtd_atual
  END,
  -- Zera qtd_nova se ela foi usada
  e.qtd_nova = CASE
    WHEN COALESCE(e.qtd_nova, 0) > 0 THEN 0
    ELSE e.qtd_nova
  END
WHERE
  e.qtd_atual <> e.qtd_virtual -- só atualiza se houver diferença
  OR COALESCE(e.qtd_nova, 0) > 0 -- ou se tiver qtd_nova pendente
  OR COALESCE(r.reservas_ativas, 0) > 0;

 -- ou se tiver reserva ativa
COMMIT;

END
```

