-- Remove titles that start with emoji/symbols, e.g. "⏰ FDA delays..."
AND NOT REGEXP_LIKE(ARTICLE_TITLE, '^[^A-Za-z0-9].*')
