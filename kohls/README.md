# Kohl's — Data Engineer (Live Coding, Python)

**Format:** live technical assessment, plain Python, **stdlib only** (pandas/numpy explicitly banned; `collections`, `datetime` encouraged).

---

## Purchase data aggregation

> You are provided with a flat list representing purchase records. The data is sequential; **every 4 elements represent a single transaction**:
>
> - SKU (string)
> - Item description (string)
> - Price (string, e.g. `'$10.00'`)
> - Purchase date (string, `'YYYY-MM-DD'`)
>
> Process the `purchases` list and print a single formatted string:
>
> `"{most_common_sku} {average_price} {min_date} {max_date}"`
>
> where `average_price` is the average price **of the most common SKU**, formatted to 2 decimal places, and min/max dates are that SKU's first and last purchase dates.

```python
purchases = [
    '123456', 'Nike T-shirt',  '$12.65', '2026-01-01',
    '194850', 'Vans Shoes',    '$89.99', '2026-01-04',
    '123456', 'Nike T-shirt',  '$12.65', '2026-01-05',
    '202110', 'Backpack',      '$45.00', '2026-01-10',
    '123456', 'Nike T-shirt',  '$15.00', '2026-01-12',
    '555432', 'Water Bottle',  '$22.50', '2026-02-01',
    '194850', 'Vans Shoes',    '$85.00', '2026-02-05',
    '123456', 'Nike T-shirt',  '$12.65', '2026-02-10',
    '999001', 'Running Socks', '$8.00',  '2026-02-15',
    '202110', 'Backpack',      '$42.00', '2026-02-20',
]
```

### Clean solution

```python
def aggregate_purchase_data(data):
    sku_info = {}
    for i in range(0, len(data), 4):
        sku = data[i]
        price = float(data[i + 2].replace('$', ''))
        date = data[i + 3]

        info = sku_info.setdefault(
            sku, {"count": 0, "total": 0.0, "min_date": date, "max_date": date}
        )
        info["count"] += 1
        info["total"] += price
        info["min_date"] = min(info["min_date"], date)
        info["max_date"] = max(info["max_date"], date)

    top = max(sku_info, key=lambda s: sku_info[s]["count"])
    info = sku_info[top]
    avg = info["total"] / info["count"]
    return f"{top} {avg:.2f} {info['min_date']} {info['max_date']}"


if __name__ == "__main__":
    print(aggregate_purchase_data(purchases))  # 123456 13.24 2026-01-01 2026-02-10
```

### What they're actually testing

- **Stride iteration** over a flat list (`range(0, len(data), 4)`) without tripping on indexes.
- String → number cleaning (`'$12.65'` → `12.65`).
- One-pass aggregation into a dict — the mental model of `GROUP BY` in plain Python.
- The nice catch: ISO `YYYY-MM-DD` dates compare correctly **as strings**, so no `datetime` parsing is needed — saying that out loud scores points.

A useful drill: solve it in SQL first (`GROUP BY sku ORDER BY COUNT(*) DESC LIMIT 1`), then translate. Interviewers at retail companies love seeing you connect the two.
