## 1.

![schema](../images/schema.png)


- **Time** (<u>TID</u>, date, month, bimester, trimester, semester, year)
- **Museum** (<u>MID</u>, museum, category, city, province, region, s1, s2, s3, s4, s5, s6, s7, s8, s9, s10)
- **Purchase_Modality** (<u>PID</u>, purchase_modality)
- **Ticket** (<u>TTID</u>, ticket_type)
- **Time_Slot** (<u>TSID</u>, time_slot)
- **Sale** (<u>TID</u>, <u>MID</u>, <u>PID</u>, <u>TTID</u>, <u>TSID</u>, num_tickets, revenue)

>S1 through S10 represent different kinds of additional services, serving as _Configuration Attributes_ that accept boolean values (Y/N).

## 2.
### 2.1
```SQL
SELECT ticket_type, 
		month,
		SUM(revenue)/COUNT(distinct date),
		SUM(SUM(revenue)) OVER (PARTITION BY year, ticket_type
                         ORDER BY month
                         ROWS UNBOUNDED PRECEDING),
		100 * SUM(num_tickets)/SUM(SUM(num_tickets)) OVER (PARTITION BY month)

FROM Sale S, Ticket TT, Time T

WHERE S.TID = T.TID AND S.TTID = TT.TTID

GROUP BY ticket_type, month, year
```
### 2.2
```SQL
SELECT museum, 
	ticket_type,
	SUM(revenue)/SUM(num_tickets),
	100 * SUM(revenue)/SUM(SUM(revenue)) OVER (PARTITION BY categoty),
	RANK() OVER (PATITION BY ticket_type
			ORDER BY SUM(num_tickets) desc)

FROM Sale S, Ticket TT, Time T, Museum M

WHERE S.TTID = TT.TTID AND S.MID = M.MID AND S.TID = T.TID AND
	year = '2021'

GROUP BY MID, museum, ticket_type, category
```

### 3.

Query a.
```SQL
SELECT ticket_type, semester,
	SUM(revenue)/COUNT(distinct month)

FROM Sale S, Ticket TT, Time T

WHERE <joins>

GROUP BY ticket_type, semester
```

Query b.
```SQL
SELECT ticket_type, month,
	SUM(SUM(revenue)) OVER (PARTITION BY year, ticket_type 
						ORDER BY month
						ROWS UNBOUNDED PRECEDING)

FROM Sale S, Time T, Ticket TT

WHERE <joins>

GROUP BY ticket_type, month, year
```

Query c.
```SQL
SELECT ticket_type, month,
	SUM(num_tickets),
	SUM(revenue),
	AVG(revenue)

FROM Sale S, Ticket TT, Time T, Purchase_Modality P

WHERE <joins> AND
	purchase_modality = 'online'

GROUP BY ticket_type, month
```

Query d.
```SQL
SELECT ticket_type, month,
	SUM(num_tickets),
	SUM(revenue),
	AVG(revenue)

FROM Sale S, Ticket TT, Time T

WHERE <joins> AND
	year = '2021'

GROUP BY ticket_type, month
```

Query e.
```SQL
SELECT ticket_type, month,
	100 * SUM(num_tickets)/SUM(SUM(num_tickets)) OVER (PARTITION BY month)

FROM Sale S, Ticket TT, Time T

WHERE <joins>

GROUP BY ticket_type, month
```

### 3.1

**Materialized View:**

```SQL
SELECT ticket_type, month, semester, year, purchase_modality, 
	SUM(revenue) AS tot_revenue
	SUM(num_tickets) AS tot_tickets

FROM Sale S, Time T, Ticket TT, Purchase_Modility P

WHERE S.TID = T.TID AND S.TTID = TT.TTID AND S.PID = P.PID

GROUP BY ticket_type, month, semester, year, purchase_modality
```
### 3.2
```SQL
CREATE MATERIALIZED VIEW LOG ON Sale
WITH SEQUENCE, ROWID 
(TTID, TID, PID, num_tickets, revenue) 
INCLUDING NEW VALUES;

CREATE MATERIALIZED VIEW LOG ON Ticket
WITH SEQUENCE, ROWID 
(TTID, ticket_type) 
INCLUDING NEW VALUES;

CREATE MATERIALIZED VIEW LOG ON Time
WITH SEQUENCE, ROWID 
(TID, month, semester, year) 
INCLUDING NEW VALUES;

CREATE MATERIALIZED VIEW LOG ON Purchase_Modality
WITH SEQUENCE, ROWID 
(PID, purchase_modality) 
INCLUDING NEW VALUES;
```
### 3.3

1. **Sale Table:**    
	- `INSERT` into the Sale table with new values for TTID, TID, PID, num_tickets, or revenue.
	- `UPDATE` on the Sale table that modifies values in TTID, TID, PID, num_tickets, or revenue.
	- `DELETE` from the Sale table.

2. **Ticket Table:**    
	- `INSERT` into the Ticket table with new values for TTID or ticket_type.
	- `UPDATE` on the Ticket table that modifies values in TTID or ticket_type.
	- `DELETE` from the Ticket table.

3. **Time Table:**    
	- `INSERT` into the Time table with new values for TID, month, semester, or year.
	- `UPDATE` on the Time table that modifies values in TID, month, semester, or year.
	- `DELETE` from the Time table.

4. **Purchase_Modality Table:**    
	- `INSERT` into the Purchase_Modality table with new values for PID or purchase_modality.
	- `UPDATE` on the Purchase_Modality table that modifies values in PID or purchase_modality.
	- `DELETE` from the Purchase_Modality table.

## 4

### 4.1
```SQL
CREATE TABLE VM1 
	(
	month             VARCHAR(20),
	semester          VARCHAR(20), 
	year              VARCHAR(20), 
	ticket_type       VARCHAR(20), 
	purchase_modality VARCHAR(20), 
	tot_revenue       INTEGER 
	CHECK (tot_revenue IS NOT NULL AND tot_revenue > 0), 
		tot_tickets       INTEGER 
	CHECK (tot_tickets IS NOT NULL AND tot_tickets > 0), 
		PRIMARY KEY (month, ticket_type, purchase_modality) 
	)
```
### 4.2
```SQL
INSERT INTO VM1 (month, semester, year, ticket_type, purchase_modality, tot_revenue, tot_tickets)
(SELECT month, semester, year, ticket_type, purchase_modility,
	sum(revenue) AS tot_revenue,
	sum(num_tickets) AS tot_tickets

FROM Sale S, Time T, Ticket TT, Purchase_Modality P

WHERE S.TID = T.TID AND S.TTID = TT.TTID AND S.PID = P.PID

GROUP BY month, semester, year, ticket_type, purchase_modality)
```
### 4.3
```SQL
CREATE TRIGGER RefreshVM1 
AFTER INSERT ON Sale
FOR EACH ROW 
DECLARE 
N number; 
varMonth, varSemester, varYear, varTicketType, varPurchaseModality VARCHAR(20); 
BEGIN

SELECT month, semester, year INTO varMonth, VarSemester, varYear
FROM Time
WHERE TID = :NEW.TID;


SELECT ticket_type INTO varTicketType 
FROM Ticket 
WHERE TTID = :NEW.TTID; 


SELECT purchase_modality INTO varPurchaseModality
FROM Purchase_Modality
WHERE PID = :NEW.PID;


SELECT COUNT(*) INTO N 
FROM VM1 
WHERE month = varMonth AND
	ticket_type = varTicketType AND 
	purchase_modality = varPurchaseModality; 


IF (N > 0) THEN 
UPDATE VM1 
SET tot_revenue = tot_revenue + :NEW.revenue, 
    tot_tickets = tot_tickets + :NEW.num_tickets 
WHERE month = varMonth AND 
	ticket_type = varTicketType AND 
	purchase_modality = varPurchaseModality; 


ELSE 
INSERT INTO VM1 (month, semester, year, ticket_type, purchase_modality, tot_revenue, tot_tickets) 
VALUES (varMonth, varSemester, varYear, varTicketType, varPurchaseModality, 
:NEW.revenue, :NEW.num_tickets); 
END IF; 
END; 
```
### 4.4
**INSERT Operation on Sale**
-  When a new record is inserted into the `Sale` table, the trigger `RefreshVM1` is fired.
- The trigger then retrieves information from related tables (`Time`, `Ticket`, `Purchase_Modality`) to determine the values for `varMonth`, `varSemester`, `varYear`, `varTicketType`, and `varPurchaseModality`.
- It checks if a corresponding record already exists in the materialized view `VM1` for the specified month, ticket type, and purchase modality.
- If a record exists, it updates the `VM1` record by incrementing `tot_revenue` and `tot_tickets`.
- If no record exists, it inserts a new record into the `VM1` materialized view with the values retrieved from related tables and the corresponding values from the newly inserted record in the `Sale` table.
