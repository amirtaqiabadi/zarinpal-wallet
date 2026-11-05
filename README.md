# کیف‌پول زرین‌پال

این skill به عامل کدنویسی یاد می‌دهد شارژ کیف‌پول را به زرین‌پال وصل کند. مدل داده دفترکل است: موجودی از آخرین ردیف تراکنش خوانده می‌شود. شروع پرداخت، کال‌بک و برگشت با قفل ردیف، امضای HMAC و تأیید مبلغ ذخیره‌شده انجام می‌شود.

برای ساخت حساب پذیرندگی از این لینک معرف استفاده کنید:

https://next.zarinpal.com/auth/register?ref_id=1tjG3k

skill خودش پرداخت انجام نمی‌دهد. بعد از نصب، در پروژهٔ خودتان از عامل بخواهید اتصال کیف‌پول به زرین‌پال را پیاده کند. آدرس وب‌سرویس، فیلدها و کدهای خطا را عامل باید از مستندات زندهٔ زرین‌پال بخواند، نه از حافظه.

## نصب

داخل پروژه:

```bash
npx skills add amirtaqiabadi/zarinpal-wallet -a cursor
```

برای همهٔ پروژه‌های همین ماشین:

```bash
npx skills add amirtaqiabadi/zarinpal-wallet -a cursor -g
```

## اتصال مستندات زرین‌پال

در Cursor از `Settings → MCP` این سرور را اضافه کنید، یا در `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "zarinpal-docs": {
      "url": "https://www.zarinpal.com/docs/mcp"
    }
  }
}
```

سرویس عمومی است و کلید نمی‌خواهد. ابزارها: `search_docs`، `get_doc`، `list_docs`، `get_error_code`. سلامت سرویس: `https://www.zarinpal.com/docs/mcp/health`.

## متغیرهای محیط

مقادیر را فقط در محیط بگذارید و در ریپو commit نکنید.

- `ZARINPAL_MERCHANT_ID`
- `ZARINPAL_SANDBOX` با مقدار `true` یا `false`
- `PAYMENT_CALLBACK_HMAC_SECRET` حداقل ۳۲ کاراکتر تصادفی
- `PAYMENT_CALLBACK_BASE_URL` آدرس عمومی API که زرین‌پال به آن برمی‌گردد

## مجوز

MIT. متن در [LICENSE](LICENSE) است.
