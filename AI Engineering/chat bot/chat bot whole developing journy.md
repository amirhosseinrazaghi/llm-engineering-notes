---

## tags: [پروژه, چت‌بات] وضعیت: 🟢 نسخه اول کار می‌کند تاریخ شروع:



# پروژه چت‌بات فارامرز — مسیر توسعه

## 🎯 هدف پروژه در یک جمله
---
---

## tags: [پروژه, چت‌بات] وضعیت: 🏁 پروژه تکمیل شد (نسخه نهایی) تاریخ شروع:

# پروژه چت‌بات فارامرز — مسیر توسعه

## 🎯 هدف پروژه در یک جمله

ساخت یک چت‌بات خط‌فرمانی که با یک مدل زبانی بزرگ (از طریق OpenRouter) صحبت می‌کند، شخصیت و اسم ثابت دارد (فارامرز)، تاریخچه کل گفتگو را به یاد می‌آورد، و مصرف توکن و هزینه‌اش را کنترل و گزارش می‌کند.

---

## 🗺️ مسیری که طی کردیم (خلاصه ماجرا)

۱. اول با گراک شروع کردیم، ولی به‌خاطر محدودیت جغرافیایی، همیشه خطای ۴۰۳ می‌گرفتیم → رفتیم سراغ OpenRouter. ۲. اولین مدلی که انتخاب کردیم اصلاً یه مدل مکالمه‌ای نبود (یه مدل تشخیص خطر بود) → فهمیدیم باید حتماً نوع مدل رو قبل از استفاده بررسی کنیم. ۳. یه باگ داشتیم که به‌جای فرستادن کل تاریخچه گفتگو، فقط پیام تازه رو می‌فرستادیم (`messages=message` به‌جای `messages=messages`) → با درست‌کردن این خط، بات حافظه واقعی پیدا کرد. ۴. مدل‌های رایگان OpenRouter گاهی شلوغ می‌شن یا از رده خارج می‌شن → یاد گرفتیم چطور با `extra_body` چند مدل پشتیبان معرفی کنیم. ۵. در نسخه اول، یه شخصیت ثابت (فارامرز) به بات دادیم با پیام سیستمی. ۶. تو نسخه دوم، کد رو به یک کلاس تبدیل کردیم و کنترل توکن و هزینه رو اضافه کردیم. ۷. تو نسخه سوم، استریم واقعی و توکنایزر درست مدل رو اضافه کردیم. ۸. تو نسخه نهایی، دو روش مدیریت حافظه اضافه شد: اول پنجره لغزان (`trim_messages`)، بعد خلاصه‌سازی با مدل (`summerize`) — جزئیات مفهومی تو [[chat bot memory management (sliding window and summarization)]].

جزئیات کامل هر کدوم از این باگ‌ها تو یادداشت [[chat bot bugs]] ثبت شده. مفاهیم توکن و معیارهای عملکرد استفاده‌شده در نسخه دوم تو [[chat bot token pipeline and performance metrics]] هست.

---

## 💻 کد کامل نسخه اول (تاریخی — برای مرجع)

```python
import dotenv
import os
from openai import OpenAI
dotenv.load_dotenv()

llm = OpenAI(
    api_key=os.getenv("OPEN_ROUTER_API_KEY"),
    base_url="https://openrouter.ai/api/v1"
)
messages = [
    {'role': 'system', 'content': '''you are a helpful assistent, your name is faramarz,
     in the first response intreduce yourself give short concise answers'''}
]


def bot(message):
    messages.append({'role': 'user', 'content': message})
    response = llm.chat.completions.create(
        model="minimax/minimax-m3:free",
        messages=messages,
        temperature=0.7,
        max_tokens=100,
    )

    response_text = response.choices[0].message.content
    messages.append({'role': 'assistant', 'content': response_text})

    return response_text


def chat():
    while True:
        user_input = input(
            'Ask Something (type "Q" to terminate)>>> ').strip().lower()

        if user_input == "q":
            print("good luck!")
            break

        bot_answer = bot(user_input)

        print(bot_answer)


def main():
    chat()
    print(messages)


if __name__ == '__main__':
    main()
```

توضیح کامل خط‌به‌خط این نسخه (تنظیمات اولیه، تابع `bot`، تابع `chat`، تابع `main`) عوض نشده و هنوز معتبره — پایین فقط **تغییرات** نسخه دوم توضیح داده شده تا تکراری نشه.

---

## 🔁 نسخه دوم: تبدیل به کلاس + کنترل توکن ورودی + محاسبه هزینه

### چرا این تغییر لازم بود

با تابع‌های جدا و متغیر `messages` سراسری، نمی‌شد دو تا مکالمه جدا از هم داشت (هر دو از یه لیست مشترک استفاده می‌کردن). تبدیل به کلاس یعنی هر شیء `ChatBot` تاریخچه و شمارنده توکن خودش رو جدا نگه می‌داره.

### کد کامل نسخه دوم

```python
import tiktoken
import dotenv
import os
from openai import OpenAI
dotenv.load_dotenv()


class ChatBot:
    def __init__(self):
        self.llm = OpenAI(
            api_key=os.getenv("OPEN_ROUTER_API_KEY"),
            base_url="https://openrouter.ai/api/v1"
        )

        self.messages = [
            {'role': 'system', 'content': '''you are a helpful assistent, your name is faramarz,
            in the first response intreduce yourself give short concise answers'''}
        ]

        self.total_token = 0
        self.input_token = 0
        self.output_token = 0

    def bot(self, message):

        self.messages.append({'role': 'user', 'content': message})
        response = self.llm.chat.completions.create(
            model="poolside/laguna-s-2.1:free",
            messages=self.messages,
            temperature=0.7,
            max_tokens=100,
        )

        response_text = response.choices[0].message.content
        self.messages.append({'role': 'assistant', 'content': response_text})

        self.total_token += response.usage.total_tokens
        self.input_token += response.usage.prompt_tokens
        self.output_token += response.usage.completion_tokens
        return response_text

    def chat(self):
        encoding = tiktoken.get_encoding('cl100k_base')

        while True:
            user_input = input(
                'Ask Something (type "Q" to terminate)>>> ').strip().lower()

            if user_input == "q":
                print("good luck!")
                break

            token_count = len(encoding.encode(user_input))

            if token_count > 10:
                print("your prompt exceeded the limit try a short one")
                continue

            bot_answer = self.bot(user_input)

            print(bot_answer)

    def report(self):
        for message in self.messages:
            print(message)
        print(f"consuend token: {self.total_token}")
        print(
            f"$ {(self.input_token * 0.05 + self.output_token * 0.08) / 1_000_000}")


def main():
    bot = ChatBot()
    bot.chat()
    bot.report()


if __name__ == '__main__':
    main()
```

### 🔍 توضیح فقط تغییرات (نسبت به نسخه اول)

**تبدیل به کلاس:**

```python
class ChatBot:
    def __init__(self):
        self.llm = ...
        self.messages = [...]
        self.total_token = 0
        self.input_token = 0
        self.output_token = 0
```

همون `llm` و `messages` نسخه اول‌ان، فقط الان به‌جای متغیر سراسری، `self.` جلوشون اومده — یعنی مال همون شیء `ChatBot` هستن، نه کل برنامه. سه‌تا شمارنده جدید هم اضافه شده تا مصرف توکن رو تجمعی نگه دارن.

**شمارش توکن بعد از درخواست، تو `bot`:**

```python
self.total_token += response.usage.total_tokens
self.input_token += response.usage.prompt_tokens
self.output_token += response.usage.completion_tokens
```

هر بار جواب می‌رسه، سرور خودش دقیقاً می‌گه چند توکن ورودی و چند توکن خروجی مصرف شده (تو فیلد `usage`)؛ این سه خط اون عددها رو جمع می‌زنه تا آخرش کل مصرف مکالمه معلوم باشه (توضیح کامل مفهومی تو [[chat bot token pipeline and performance metrics]]).

**شمارش توکن قبل از درخواست، تو `chat`:**

```python
encoding = tiktoken.get_encoding('cl100k_base')
...
token_count = len(encoding.encode(user_input))

if token_count > 10:
    print("your prompt exceeded the limit try a short one")
    continue
```

قبل از اینکه پیام کاربر اصلاً به مدل فرستاده بشه، محلی (بدون اتصال به سرور) شمرده می‌شه چند توکنه. اگه بیش از ۱۰ توکن بود، `continue` باعث می‌شه بدون فراخوانی `bot()` دوباره برگرده اول حلقه و از کاربر پیام بگیره — یعنی حتی یه درخواست هم به سرور فرستاده نمی‌شه و هزینه‌ای هم خرج نمی‌شه.

⚠️ **نکته‌ای که باید بدونی:** `cl100k_base` توکنایزر رسمی OpenAI هست، نه توکنایزر واقعی مدل `poolside/laguna-s-2.1` که اینجا صداش می‌زنی. یعنی این عدد فقط یه تخمینه، نه دقیق. جزئیات و راه‌حل تو [[chat bot token pipeline and performance metrics]] بخش «شمارش توکن قبل از درخواست».

**متد `report` (جدید):**

```python
def report(self):
    for message in self.messages:
        print(message)
    print(f"consuend token: {self.total_token}")
    print(
        f"$ {(self.input_token * 0.05 + self.output_token * 0.08) / 1_000_000}")
```

جایگزین خط تنهای `print(messages)` نسخه اول شده. اول کل تاریخچه گفتگو رو چاپ می‌کنه، بعد جمع کل توکن مصرفی، و در آخر هزینه تقریبی دلاری بر اساس فرمول `(ورودی × قیمت‌ورودی + خروجی × قیمت‌خروجی) / 1,000,000` (فرمول کامل تو [[chat bot token pipeline and performance metrics]] بخش «محاسبه هزینه»).

**تو `main`:**

```python
def main():
    bot = ChatBot()
    bot.chat()
    bot.report()
```

به‌جای صدازدن مستقیم تابع‌ها، اول یه شیء از کلاس ساخته می‌شه، بعد گفتگو شروع می‌شه، و بعد از تمومِ گفتگو گزارش چاپ می‌شه.

---

## 🔁 نسخه سوم: استریم واقعی + توکنایزر درست مدل + سقف روزانه

### چرا این تغییر لازم بود

نسخه دوم دو تا مشکل باز داشت که خودمون تو «مشکل بعدی» ثبت کرده بودیم: شمارش توکن با توکنایزر اشتباه بود، و جواب‌ها یک‌جا چاپ می‌شدن نه زنده. نسخه سوم دقیقاً همین دوتا رو حل کرده، به‌علاوه یه سقف مصرف روزانه اضافه شده.

### کد کامل نسخه سوم

```python
import dotenv
import time
import os
from openai import OpenAI
from transformers import AutoTokenizer
dotenv.load_dotenv()


class ChatBot:
    def __init__(self):
        self.llm = OpenAI(
            api_key=os.getenv("OPEN_ROUTER_API_KEY"),
            base_url="https://openrouter.ai/api/v1"
        )

        self.messages = [
            {'role': 'system', 'content': '''you are a helpful assistent, your name is faramarz,
            in the first response intreduce yourself give short concise answers'''}
        ]

        self.total_token = 0
        self.input_token = 0
        self.output_token = 0

    def bot(self, message):

        self.messages.append({'role': 'user', 'content': message})
        response = self.llm.chat.completions.create(
            model="poolside/laguna-s-2.1:free",
            messages=self.messages,
            temperature=0.7,
            max_tokens=100,
            stream=True
        )

        # for streaming the tokens =============================================================
        response_text = ''
        for chunk in response:
            if chunk.choices:

                delta = chunk.choices[0].delta.content

                if delta:
                    print(delta, end="", flush=True)
                    response_text += delta
                    time.sleep(0.05)
# ============================================================================================

# in token streaming we get the usage after the last chunk====================================
            if chunk.usage:
                self.total_token += chunk.usage.total_tokens
                self.input_token += chunk.usage.prompt_tokens
                self.output_token += chunk.usage.completion_tokens

        print()
        self.messages.append({'role': 'assistant', 'content': response_text})

    def chat(self):
        encoding = AutoTokenizer.from_pretrained('poolside/Laguna-S-2.1')

        while True:
            user_input = input(
                'Ask Something (type "Q" to terminate)>>> ').strip().lower()

            if user_input == "q":
                print("good luck!")
                break

            token_count = len(encoding.encode(user_input))

            if token_count > 10:
                print("your prompt exceeded the limit try a short one")
                continue

            if self.total_token > 300:
                print("unfortunaly you exceded the total token limit for today! ")
                break

            self.bot(user_input)

    def report(self):
        for message in self.messages:
            print(message)
        print(f"consuend token: {self.total_token}")
        print(
            f"$ {(self.input_token * 0.05 + self.output_token * 0.08) / 1_000_000}")


def main():
    bot = ChatBot()
    bot.chat()
    bot.report()


if __name__ == '__main__':
    main()
```

### 🔍 توضیح فقط تغییرات (نسبت به نسخه دوم)

**تعویض توکنایزر، تو `chat`:**

```python
from transformers import AutoTokenizer
...
encoding = AutoTokenizer.from_pretrained('poolside/Laguna-S-2.1')
```

به‌جای `tiktoken` با انکودینگ عمومی OpenAI، الان مستقیماً توکنایزر خودِ مدل `poolside/Laguna-S-2.1` از HuggingFace دانلود و استفاده می‌شه — یعنی شمارش توکن قبل از درخواست دیگه تخمین نیست، دقیقه (جزئیات کامل تو [[chat bot token pipeline and performance metrics]]).

**فعال‌کردن استریم، تو `bot`:**

```python
response = self.llm.chat.completions.create(
    ...,
    stream=True
)

response_text = ''
for chunk in response:
    if chunk.choices:
        delta = chunk.choices[0].delta.content
        if delta:
            print(delta, end="", flush=True)
            response_text += delta
            time.sleep(0.05)
```

به‌جای گرفتن یه جواب کامل و یک‌جا، الان یه حلقه از تکه‌های کوچیک (chunk) می‌گیریم. هر چانک ممکنه یه تکه از متن جواب (`delta`) داشته باشه یا نه؛ اگه داشت، همون لحظه چاپ می‌شه (`flush=True`) و به `response_text` هم اضافه می‌شه تا در آخر کل جواب کامل رو داشته باشیم و بتونیم تو تاریخچه ذخیره‌ش کنیم. `time.sleep(0.05)` فقط یه مکث مصنوعی کوچیکه تا جلوه‌ی تایپ‌شدن قابل‌دیدن‌تر باشه (بدون این خط، تکه‌ها احتمالاً به‌قدری سریع می‌رسیدن که همچنان یهویی به نظر می‌رسید). توضیح کامل مفهوم استریم، `delta`، و `flush` تو [[chat bot token pipeline and performance metrics]] بخش ۸.

**گرفتن `usage` بعد از آخرین چانک:**

```python
if chunk.usage:
    self.total_token += chunk.usage.total_tokens
    self.input_token += chunk.usage.prompt_tokens
    self.output_token += chunk.usage.completion_tokens
```

تو حالت استریم، عدد مصرف توکن فقط تو آخرین چانک می‌رسه، برای همین این چک جدا از چک `delta` نوشته شده — تا زمانی که چانکی `usage` نداره، این بخش کاری نمی‌کنه.

**سقف مصرف روزانه، تو `chat`:**

```python
if self.total_token > 300:
    print("unfortunaly you exceded the total token limit for today! ")
    break
```

قبل از فرستادن هر پیام جدید، چک می‌شه که آیا کل توکن مصرفی این مکالمه (که تو `bot()` هر بار جمع زده می‌شه) از ۳۰۰ رد شده یا نه. اگه رد شده بود، `break` کل حلقه گفتگو رو متوقف می‌کنه — یعنی برخلاف چک ۱۰ توکنی (که فقط همون یه پیام رو رد می‌کنه و `continue` می‌زنه)، این یکی کل مکالمه رو تموم می‌کنه.

### ارتباط با مفاهیم

جزئیات کامل استریم، تفاوت TTFT قبل و بعد این نسخه، و توضیح `flush` تو [[chat bot token pipeline and performance metrics]] هست.

---

## 🔁 نسخه نهایی: مدیریت حافظه با پنجره لغزان + خلاصه‌سازی

### چرا این تغییر لازم بود

هرچه مکالمه طولانی‌تر می‌شد، `self.messages` بی‌نهایت بزرگ‌تر می‌شد — هم هزینه هر درخواست بالاتر می‌رفت، هم بالاخره از سقف context window مدل رد می‌شدیم. این نسخه دو راه‌حل متفاوت رو امتحان کرد (جزئیات مفهومی کامل تو [[chat bot memory management (sliding window and summarization)]]).

### کد کامل نسخه نهایی

```python
import dotenv
import time
import os
from openai import OpenAI
from transformers import AutoTokenizer
dotenv.load_dotenv()


class ChatBot:
    def __init__(self):
        self.llm = OpenAI(
            api_key=os.getenv("OPEN_ROUTER_API_KEY"),
            base_url="https://openrouter.ai/api/v1"
        )

        self.messages = [
            {'role': 'system', 'content': '''you are a helpful assistent, your name is faramarz,
            in the first response intreduce yourself give short concise answers'''}
        ]

        self.total_token = 0
        self.input_token = 0
        self.output_token = 0
        self.iteration = 0

    def trim_messages(self, max_number=4):
        if len(self.messages) < max_number:
            return

        system_messages = [m for m in self.messages if m['role'] == 'system']
        other_messages = [m for m in self.messages if m['role'] != 'system']

        trimed_messages = other_messages[-max_number:]

        self.messages = [*system_messages, *trimed_messages]

    def summerize(self):
        if self.iteration % 5 != 0:
            return

        system_messages = [m for m in self.messages if m['role'] == 'system']
        other_messages = [m for m in self.messages if m['role'] != 'system']

        early_messages, last_messages = (
            other_messages[0:5], other_messages[5:])

        response = self.llm.chat.completions.create(
            model="inclusionai/ling-3.0-flash-fin:free",
            temperature=0.5,
            messages=[
                {'role': 'system', 'content': 'summerize thease messages concisely, keeping key  facts and information'},
                *early_messages
            ]
        )

        summerized_message = {
            'role': 'system',
            'content': f'this is summey of 5 early messages {response.choices[0].message.content}'
        }

        self.messages = [*system_messages, summerized_message, *last_messages]
        print("summerization was succesfull")

    def bot(self, message):

        self.messages.append({'role': 'user', 'content': message})
        response = self.llm.chat.completions.create(
            model="inclusionai/ling-3.0-flash-fin:free",
            messages=self.messages,
            temperature=0.7,
            stream=True
        )

        # for streaming the tokens =============================================================
        response_text = ''
        for chunk in response:
            if chunk.choices:

                delta = chunk.choices[0].delta.content

                if delta:
                    print(delta, end="", flush=True)
                    response_text += delta
                    time.sleep(0.05)
# ============================================================================================

# in token streaming we get the usage after the last chunk====================================
            if chunk.usage:
                self.total_token += chunk.usage.total_tokens
                self.input_token += chunk.usage.prompt_tokens
                self.output_token += chunk.usage.completion_tokens

        print()
        self.messages.append({'role': 'assistant', 'content': response_text})

        self.summerize()
        return response_text

    def chat(self):
        encoding = AutoTokenizer.from_pretrained(
            'inclusionAI/Ling-3.0-flash-Fin')

        while True:
            self.iteration += 1

            user_input = input(
                'Ask Something (type "Q" to terminate)>>> ').strip().lower()

            if user_input == "q":
                print("good luck!")
                break

            token_count = len(encoding.encode(user_input))

            if token_count > 10:
                print("your prompt exceeded the limit try a short one")
                continue

            if self.total_token > 10000:
                print("unfortunaly you exceded the total token limit for today! ")
                break

            self.bot(user_input)

    def report(self):
        for message in self.messages:
            print(message)
        print(f"consuend token: {self.total_token}")
        print(
            f"$ {(self.input_token * 0.05 + self.output_token * 0.08) / 1_000_000}")


def main():
    bot = ChatBot()
    bot.chat()
    bot.report()


if __name__ == '__main__':
    main()
```

### 🔍 توضیح فقط تغییرات (نسبت به نسخه سوم)

**شمارنده نوبت، تو `__init__` و `chat`:**

```python
self.iteration = 0
...
self.iteration += 1
```

هر بار حلقه‌ی `chat()` یه دور می‌زنه (یعنی کاربر یه چیزی تایپ می‌کنه)، این عدد یکی زیاد می‌شه. این شمارنده مبنای تصمیم‌گیری برای اینه که «خلاصه‌سازی» کِی اجرا بشه.

**متد `trim_messages` (پنجره لغزان):**

این متد عیناً همون منطق قبلیه (توضیح کاملش تو باگ ۹ فایل [[chat bot bugs]] اومده)، ولی نکته مهم اینه که **دیگه از داخل `bot()` صدا زده نمی‌شه** — الان تو کد فقط تعریف شده، ولی هیچ‌جا فراخوانی نمی‌شه. یعنی الان عملاً فقط از روش دوم (خلاصه‌سازی) استفاده می‌کنیم، نه هردو با هم.

**متد `summerize` (خلاصه‌سازی، جدید):**

```python
def summerize(self):
    if self.iteration % 5 != 0:
        return
```

فقط هر ۵ نوبت یه‌بار (وقتی باقیمانده‌ی تقسیم بر ۵ صفر بشه) ادامه می‌ده؛ در غیر این صورت فوراً برمی‌گرده و کاری نمی‌کنه.

```python
    system_messages = [m for m in self.messages if m['role'] == 'system']
    other_messages = [m for m in self.messages if m['role'] != 'system']

    early_messages, last_messages = (
        other_messages[0:5], other_messages[5:])
```

پیام‌های سیستمی رو جدا نگه می‌داره (دست‌نخورده می‌مونن)، بعد از بین پیام‌های عادی (کاربر و بات)، ۵ تای اول رو به‌عنوان «قدیمی» و بقیه رو به‌عنوان «اخیر» جدا می‌کنه.

```python
    response = self.llm.chat.completions.create(
        model="inclusionai/ling-3.0-flash-fin:free",
        temperature=0.5,
        messages=[
            {'role': 'system', 'content': 'summerize thease messages concisely, keeping key  facts and information'},
            *early_messages
        ]
    )
```

یه درخواست کاملاً **جدا** به مدل می‌فرسته: فقط یه دستور سیستمی («این‌ها رو خلاصه کن») به‌همراه همون ۵ پیام قدیمی — نه کل تاریخچه.

```python
    summerized_message = {
        'role': 'system',
        'content': f'this is summey of 5 early messages {response.choices[0].message.content}'
    }

    self.messages = [*system_messages, summerized_message, *last_messages]
```

جواب مدل (خلاصه) رو تو یه پیام جدید با نقش `system` می‌ذاره، و تاریخچه رو بازمی‌سازه: پیام‌های سیستمی قبلی + این خلاصه‌ی تازه + پیام‌های اخیر.

**فراخوانی `summerize` به‌جای `trim_messages`، تو `bot`:**

```python
self.summerize()
```

**سقف مصرف روزانه بالاتر رفت:** از ۳۰۰ به `10000` تغییر کرد — عدد واقعی‌تری برای یه مکالمه واقعی.

### ⚠️ نکاتی که باید بدونی (جزئیات کامل تو [[chat bot memory management (sliding window and summarization)]])

- یه باگ واقعی تو خط `other_messages[5:0]` بود که باعث می‌شد `last_messages` همیشه خالی باشه و پیام‌های اخیر پاک بشن — رفع‌شده تو [[chat bot bugs]] باگ ۱۰، با تغییر به `other_messages[5:]`.
- چون خلاصه‌ها با نقش `system` ذخیره می‌شن، هر بار خلاصه‌سازی اجرا بشه یه پیام سیستمی **اضافه** می‌شه، نه جایگزین قبلی — یعنی خودِ خلاصه‌ها می‌تونن با گذر زمان انباشته بشن.

---

## 🧩 مفاهیمی که در طول این پروژه استفاده کردیم

|مفهوم|نقشش در این کد|یادداشت مرتبط|
|---|---|---|
|کلید API و متغیر محیطی|نگه‌داشتن اطلاعات محرمانه (کلید) خارج از خود کد، در فایل `.env`|—|
|مدل زبانی بزرگ (LLM)|«مغز» چت‌بات — همون چیزی که واقعاً به پیام‌ها جواب می‌ده|[[chat bot theory]]|
|پیام سیستمی (system prompt)|تعیین شخصیت، اسم، و قوانین رفتاری ثابت بات (اینجا: فارامرز، جواب‌های کوتاه)|—|
|تاریخچه گفتگو / حافظه|نگه‌داشتن لیست کامل پیام‌ها و فرستادنش در هر درخواست، تا مدل زمینه قبلی گفتگو رو بفهمه|[[چک‌پوینت‌ها و حافظه]]|
|دما (temperature)|کنترل میزان خلاقیت در برابر دقت جواب‌ها|[[chat bot theory]]|
|سقف توکن خروجی (max_tokens)|جلوگیری از جواب‌های خیلی طولانی|[[chat bot theory]]|
|توکن (Token)|واحد اندازه‌گیری طول ورودی/خروجی که هزینه و محدودیت‌ها بر اساسش حساب می‌شه|[[chat bot theory]]|
|مدل‌های جایگزین (fallback models)|راه‌حلی برای وقتی مدل رایگان اصلی شلوغ یا از رده خارج می‌شه|[[chat bot bugs]]|
|شمارش توکن قبل/بعد از درخواست|کنترل هزینه و محدودیت طول ورودی، قبل از خرج‌کردن و بعد از دریافت جواب|[[chat bot token pipeline and performance metrics]]|
|محاسبه هزینه|تبدیل تعداد توکن مصرفی به هزینه دلاری واقعی|[[chat bot token pipeline and performance metrics]]|
|استریم (Streaming)|نمایش زنده و کلمه‌به‌کلمه‌ی جواب، به‌جای چاپ یک‌جا در آخر|[[chat bot token pipeline and performance metrics]]|
|توکنایزر واقعی مدل (AutoTokenizer)|شمارش دقیق توکن به‌جای تخمین با توکنایزر یه مدل دیگه|[[chat bot token pipeline and performance metrics]]|
|پنجره لغزان (Sliding Window)|نگه‌داشتن فقط جدیدترین پیام‌ها و دورانداختن کامل قدیمی‌ها|[[chat bot memory management (sliding window and summarization)]]|
|خلاصه‌سازی حافظه (Summarization)|فشرده‌کردن پیام‌های قدیمی با یه درخواست جدا به مدل، به‌جای دورانداختن کامل|[[chat bot memory management (sliding window and summarization)]]|

---

## 🚀 ایده‌های توسعه آینده (پروژه فعلاً تکمیل شده، این‌ها برای نسخه‌های بعدی یا پروژه‌های بعدیه)

- هنوز TTFT، TPOT و Latency واقعاً با `time.time()` اندازه‌گیری نمی‌شن.
- تاریخچه گفتگو (`self.messages`) هر بار که برنامه بسته می‌شه از بین می‌ره — برای نگه‌داشتنش بین اجراهای مختلف باید سراغ ذخیره‌سازی دائمی برم، دقیقاً همون بحث [[چک‌پوینت‌ها و حافظه]].
- `trim_messages` تعریف شده ولی دیگه صدا زده نمی‌شه — باید تصمیم بگیرم: کامل حذفش کنم، یا واقعاً هر دو روش (پنجره لغزان + خلاصه‌سازی) رو با هم ترکیب کنم.
- خلاصه‌ها (با نقش `system`) با گذر زمان انباشته می‌شن، ادغام نمی‌شن — برای مکالمه‌های خیلی طولانی باید هر خلاصه‌ی جدید رو با خلاصه‌ی قبلی ادغام کنم، نه کنارش اضافه کنم.
- مدل خلاصه‌ساز همون مدل اصلیه؛ می‌تونم بعداً از یه مدل ارزون‌تر و سبک‌تر فقط برای خلاصه‌سازی استفاده کنم تا هزینه کمتر بشه.
- `self.iteration` حتی وقتی پیام کاربر به‌خاطر محدودیت توکن رد می‌شه هم افزایش پیدا می‌کنه — یعنی شمارش نوبت‌ها دقیقاً برابر تعداد رفت‌وبرگشت‌های موفق نیست.

## 🔗 یادداشت‌های مرتبط

- [[chat bot bugs]]
- [[chat bot theory]]
- [[chat bot token pipeline and performance metrics]]
- [[chat bot memory management (sliding window and summarization)]]
- [[چک‌پوینت‌ها و حافظه]]