# HCSC — Senior ML Engineer (Live Coding)

**Format:** live coding over video call, shared editor. SQL first, then Python. Practical difficulty — LeetCode-easy/medium — but they watch how you structure and explain.

---

## SQL — Salary above department average

> Find the names of employees whose salary is higher than the average salary of departments with more than 3 employees.

```
employees                      departments
+---------------+---------+    +---------------+---------+
| emp_id        | int     |    | department_id | int     |
| name          | varchar |    | dept_name     | varchar |
| department_id | int     |    +---------------+---------+
| salary        | int     |
+---------------+---------+
```

```sql
WITH dept_stats AS (
  SELECT department_id,
         AVG(salary) AS avg_salary,
         COUNT(*)    AS emp_count
  FROM employees
  GROUP BY department_id
  HAVING COUNT(*) > 3
)
SELECT e.name
FROM employees e
JOIN dept_stats d
  ON e.department_id = d.department_id
WHERE e.salary > d.avg_salary;
```

Talking points they probed: why `HAVING` not `WHERE`, and the correlated-subquery alternative vs the CTE (CTE reads better, one scan for the stats).

---

## Python — Valid parentheses

> Given a string `s` containing only `( ) { } [ ]`, determine if the input string is valid: open brackets closed by the same type, in the correct order, every closing bracket has an opener. Return `True`/`False`.

```python
def is_valid(s: str) -> bool:
    pairs = {')': '(', '}': '{', ']': '['}
    stack = []
    for ch in s:
        if ch in pairs:
            if not stack or stack.pop() != pairs[ch]:
                return False
        else:
            stack.append(ch)
    return not stack
```

O(n) time, O(n) space. Classic stack question — say the invariant out loud: *the stack always holds the currently-unclosed openers, innermost on top.*

---

## Python — Flatten nested JSON

> Flatten the given JSON data.

```python
data = {
    "id": 1,
    "name": "John Doe",
    "contact": {
        "email": "john.doe@example.com",
        "phone": {"home": "123-456-7890", "mobile": "987-654-3210"}
    },
    "address": {"city": "New York", "zip": "10001"}
}

# Expected output:
# {
#   'id': 1,
#   'name': 'John Doe',
#   'contact_email': 'john.doe@example.com',
#   'contact_phone_home': '123-456-7890',
#   'contact_phone_mobile': '987-654-3210',
#   'address_city': 'New York',
#   'address_zip': '10001'
# }
```

```python
def flatten(d: dict, parent: str = "", sep: str = "_") -> dict:
    out = {}
    for k, v in d.items():
        key = f"{parent}{sep}{k}" if parent else k
        if isinstance(v, dict):
            out.update(flatten(v, key, sep))
        else:
            out[key] = v
    return out
```

The most DE-flavored Python question there is — it shows up everywhere. Know the recursive version cold, and be ready for the follow-up: *what about lists inside the JSON?* (index into the key: `contact_phones_0`).

---

## Takeaways

- The whole round is fundamentals under observation: window-free SQL aggregation, a stack, and recursion. Nothing exotic.
- Narrate as you go. Silence while typing is the failure mode, not wrong syntax.
