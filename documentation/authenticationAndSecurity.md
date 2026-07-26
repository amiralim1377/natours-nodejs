# راهنمای جامع مدیریت خطا و احراز هویت در Node.js و TypeScript

## مقدمه

این جزوه آموزشی، راهنمای گام‌به‌گام و معماری‌محور برای پیاده‌سازی سیستم مدیریت خطای سراسری و احراز هویت (Authentication) با استفاده از JSON Web Token (JWT) در فریم‌ورک Express و محیط TypeScript است. مطالب این جزوه با رویکرد مهندسی نرم‌افزار و توجه به اصول امنیتی تنظیم شده است و شما را از مفاهیم پایه تا پیاده‌سازی‌های پیشرفته همراهی می‌کند.

---

## فصل اول: مدیریت جامع خطاها در Express

در برنامه‌های توسعه‌یافته با Express، مدیریت خطاها نباید به صورت پراکنده در کنترلرهای مختلف انجام شود. بهترین الگو، ایجاد یک مسیر متمرکز و یک «میدل‌ور مدیریت خطای سراسری» (Global Error Handler) است.

### ساختار میدل‌ور مدیریت خطا

این میدل‌ور، تمامی خطاهایی که در طول چرخه حیات درخواست رخ می‌دهند را دریافت کرده و بر اساس محیط توسعه (Development) یا تولید (Production) پاسخ مناسبی را به کاربر ارسال می‌کند.

در محیط Production، تنها خطاهای عملیاتی (Operational) که توسط خود برنامه‌نویس پیش‌بینی شده‌اند به کاربر نمایش داده می‌شوند و خطاهای سیستمی و باگ‌ها به منظور حفظ امنیت، مخفی می‌مانند.

```typescript
import { Request, Response, NextFunction } from "express";
import { config } from "../config/env";
import { AppError } from "../utils/AppError";

export const globalErrorHandler = (
  err: any,
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  err.statusCode = err.statusCode || 500;
  err.status = err.status || "error";

  if (config.nodeEnv === "development") {
    res.status(err.statusCode).json({
      status: err.status,
      error: err,
      message: err.message,
      stack: err.stack,
    });
  } else {
    if (err.isOperational) {
      res.status(err.statusCode).json({
        status: err.status,
        message: err.message,
      });
    } else {
      res.status(500).json({
        status: "error",
        message: "Something went wrong",
      });
    }
  }
};
```

> **نکته:** در پروژه‌های TypeScript، نوع متغیر `err` در میدل‌ور خطای سراسری معمولاً به جای کلاس `AppError` برابر با `any` در نظر گرفته می‌شود. دلیل این امر این است که ممکن است خطاهای خارجی (مانند خطاهای ارتباط با پایگاه داده Mongoose یا خطاهای استاندارد جاوااسکریپت) وارد این تابع شوند که از کلاس `AppError` ارث‌بری نکرده‌اند و فیلدهایی نظیر `statusCode` در آن‌ها وجود ندارد.

### اتصال مدیریت خطا سراسری-globalErrorhandler به جریان اصلی برنامه

برای فعال‌سازی این سیستم، باید میدل‌ور خطا را به عنوان **آخرین ایستگاه** در جریان برنامه (فایل `app.ts`) ثبت کنید. همچنین باید مسیریابی پیش‌فرض برای درخواست‌هایی که به هیچ روت مشخصی متصل نمی‌شوند (مدیریت خطای 404) تعبیه گردد.

```typescript
import express, { Express } from "express";
import userRouter from "./routes/user.routes";
import { config } from "./config/env";
import morgan from "morgan";
import { AppError } from "./utils/AppError";
import { globalErrorHandler } from "./middlewares/errorHandler";

const app: Express = express();

if (config.nodeEnv === "development") {
  app.use(morgan("dev"));
}

app.use(express.json());

app.use("/api/v2/users", userRouter);

// مدیریت مسیرهای یافت‌نشده
app.all("*", (req, res, next) => {
  next(new AppError(`Cannot find ${req.originalUrl}`, 404));
});

// ثبت میدل‌ور خطای سراسری در انتهای زنجیره
app.use(globalErrorHandler);

export default app;
```

> **خلاصه فصل اول:** مدیریت متمرکز خطاها به کمک یک میدل‌ور اختصاصی در انتهای زنجیره درخواست‌های Express انجام می‌شود. این الگو، امکان تفکیک پاسخ‌های خطا در محیط‌های توسعه و تولید را فراهم کرده و امنیت برنامه را افزایش می‌دهد.

**ادامه مسیر یادگیری:** پس از ایمن‌سازی سرور در برابر خطاها، گام بعدی ایمن‌سازی دسترسی‌ها و شناخت مفاهیم احراز هویت است.

---

## فصل دوم: مبانی احراز هویت و JSON Web Token (JWT)

پروتکل HTTP ذاتاً «بدون وضعیت» (Stateless) است؛ به این معنا که هیچ حافظه‌ای برای به خاطر سپردن کاربران در درخواست‌های متوالی ندارد. بنابراین، نیازمند سازوکاری هستیم تا کاربران پس از ورود، هویت خود را در درخواست‌های بعدی اثبات کنند.

### تقابل معماری Stateful و Stateless

1. **روش قدیمی (Stateful):** سرور پس از ورود کاربر، اطلاعات او را در حافظه (Session) ذخیره کرده و یک شناسه (Cookie) به کاربر می‌دهد. این روش با افزایش تعداد کاربران، منابع سرور را به شدت درگیر کرده و در سیستم‌های توزیع‌شده (Microservices) چالش‌برانگیز است.
2. **روش مدرن (Stateless):** سرور با استفاده از JWT یک «پاسپورت رمزنگاری‌شده» تولید می‌کند. اطلاعات وضعیت کاربر درون این توکن قرار دارد و سرور نیازی به ذخیره اطلاعات در حافظه خود ندارد. این رویکرد مقیاس‌پذیری بالایی به همراه دارد.

### کالبدشکافی ساختار JWT

یک توکن JWT از سه بخش مجزا تشکیل می‌شود که با نقطه (`.`) از یکدیگر جدا می‌گردند:

1. **سربرگ (Header):** نوع توکن و الگوریتم رمزنگاری (مانند HS256) را مشخص می‌کند.
2. **بار مفید (Payload):** داده‌هایی که قصد انتقال آن‌ها را داریم (مانند ID کاربر، تاریخ صدور و تاریخ انقضا). این بخش رمزنگاری نمی‌شود و با فرمت Base64 کدگذاری می‌گردد؛ در نتیجه برای همگان قابل خواندن است و هرگز نباید حاوی اطلاعات حساس مانند رمز عبور باشد.
3. **امضای دیجیتال (Signature):** قلب تپنده و عامل امنیت توکن است. سرور با استفاده از یک کلید محرمانه (Secret Key)، بخش‌های Header و Payload را با یکدیگر ترکیب کرده و امضا را می‌سازد.

> **نکته امنیتی (Cryptographic Verification):** اگر فردی سعی کند داده‌های درون Payload را تغییر دهد (مثلاً سطح دسترسی خود را به ادمین ارتقا دهد)، از آنجایی که کلید محرمانه سرور را در اختیار ندارد، نمی‌تواند امضای دیجیتال معتبری برای داده‌های جدید بسازد. در نتیجه، سرور در کسری از ثانیه متوجه دستکاری شدن توکن شده و آن را رد می‌کند.

> **خلاصه فصل دوم:** ساختار بدون وضعیت JWT مشکل مقیاس‌پذیری را در سیستم‌های مبتنی بر HTTP برطرف می‌کند. امنیت این توکن‌ها صرفاً متکی به مخفی ماندن و قدرت کلید محرمانه (Secret Key) است.

**ادامه مسیر یادگیری:** با درک تئوری JWT، اکنون آماده پیاده‌سازی فرآیند ثبت‌نام و تولید اولین توکن در سیستم هستیم.

---

## فصل سوم: پیاده‌سازی ثبت‌نام و تولید توکن

فرآیند استاندارد در سیستم‌های مدرن این است که پس از ثبت‌نام موفقیت‌آمیز، کاربر فوراً احراز هویت شده و نیازی به ورود مجدد نداشته باشد. برای این منظور، سرور باید همزمان با ثبت کاربر در پایگاه داده، یک توکن معتبر صادر نماید.

### متغیرهای محیطی

پیش از آغاز کدنویسی، باید کلید محرمانه و زمان انقضای توکن در متغیرهای محیطی سیستم تعریف شوند. استفاده از زمان انقضا ضروری است، زیرا توکن‌های بدون انقضا در صورت افشا شدن، همواره خطرساز خواهند بود.

### پیاده‌سازی تابع تولید توکن

بر اساس اصل DRY (تکرار پرهیز)، فرآیند تولید توکن باید در یک تابع کمکی متمرکز شود تا هم در زمان ثبت‌نام و هم در زمان ورود قابل استفاده باشد.

> **توجه:** درون Payload توکن تنها شناسه یکتای کاربر (`_id`) قرار داده می‌شود. استفاده از داده‌های متغیر (مانند نام یا ایمیل) توصیه نمی‌شود؛ زیرا در صورت تغییر این اطلاعات توسط کاربر، توکنِ صادر شده حاوی اطلاعات منسوخ خواهد بود. علاوه بر این، کاهش داده‌های Payload باعث کاهش حجم توکن و افزایش سرعت شبکه می‌شود.

```typescript
import jwt from "jsonwebtoken";
import { config } from "../config/env";

export const signToken = (id: string): string => {
  return jwt.sign({ id }, config.jwtSecret, {
    expiresIn: config.jwtExpiresIn as any,
  });
};
```

### کنترلر ثبت‌نام (Signup)

مراحل پیاده‌سازی کنترلر ثبت‌نام:

1. دریافت اطلاعات از درخواست.
2. ایجاد کاربر جدید در دیتابیس.
3. تولید توکن برای کاربر جدید.
4. پاک‌سازی رمز عبور از شیء پاسخ (برای جلوگیری از افشای آن).
5. ارسال توکن و اطلاعات کاربر به عنوان پاسخ موفقیت‌آمیز.

```typescript
import { Request, Response, NextFunction } from "express";
import User from "../models/userModel";
import { signToken } from "../utils/auth"; // مسیر تابع کمکی

export const signUp = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  const { name, password, email } = req.body;
  const newUser = await User.create({ name, password, email });

  newUser.password = undefined;
  const token = signToken(newUser._id.toString());

  res.status(201).json({
    status: "success",
    message: "user signup successfully",
    token,
    data: {
      user: newUser,
    },
  });
};
```

> **خلاصه فصل سوم:** صدور توکن پس از ثبت‌نام باعث بهبود تجربه کاربری می‌شود. قرار دادن حداقل اطلاعات ممکن (مانند شناسه کاربر) در توکن، پایداری داده‌ها و بهینگی حجم ترافیک شبکه را تضمین می‌کند.

**ادامه مسیر یادگیری:** پس از پیاده‌سازی ثبت‌نام، باید فرآیند پیچیده‌ترِ ورود (Login) و مقایسه امن رمزهای عبور را در سمت پایگاه داده بررسی کنیم.

---

## فصل چهارم: ورود به سیستم (Login) و متدهای Mongoose

فرآیند ورود نیازمند دریافت اطلاعات از کاربر، جستجو در دیتابیس و بررسی صحت رمز عبور است.

### اصول امنیتی ورود

1. **جلوگیری از تشخیص کاربران (User Enumeration Prevention):** در صورت اشتباه بودن ایمیل یا رمز عبور، همواره باید یک پیام خطای عمومی و یکسان (مانند "ایمیل یا رمز عبور اشتباه است") ارسال شود تا هکرها نتوانند از روی نوع خطای بازگشتی، آدرس‌های ایمیلِ ثبت‌شده در سیستم را شناسایی کنند.
2. **استفاده از توابع Async در بررسی رمز:** مقایسه رمزها توسط الگوریتم Bcrypt نیازمند پردازش سنگین است تا در برابر حملات Brute-Force مقاومت کند. استفاده از ساختار ناهمگام (Async) از مسدود شدن پردازشگر اصلی Node.js جلوگیری می‌کند.

### توسعه Model با متدهای اختصاصی (Instance Methods)

برای جلوگیری از پراکندگی منطق برنامه، عملیات مربوط به پایگاه داده باید درون خود مدل‌ها صورت گیرد. ما یک متد برای مقایسه رمز عبور ورودی با رمز عبور هش‌شده در دیتابیس می‌نویسیم.

```typescript
import { Schema, model, Document } from "mongoose";
import bcrypt from "bcrypt";

export interface IUserMethods extends Document {
  correctPassword(
    candidatePassword: string,
    userPassword: string,
  ): Promise<boolean>;
  changedPasswordAfter(JWTTimestamp: number): boolean;
}

const userSchema = new Schema<
  IUser,
  Model<IUser, {}, IUserMethods>,
  IUserMethods
>(
  {
    name: { type: String, required: true },
    email: { type: String, required: true, unique: true },
    password: { type: String, required: true, select: false },
    // سایر فیلدها...
  },
  { timestamps: true },
);

// متد مقایسه رمز عبور
userSchema.methods.correctPassword = async function (
  candidatePassword: string,
  userPassword: string,
): Promise<boolean> {
  return await bcrypt.compare(candidatePassword, userPassword);
};

// متد بررسی تغییر رمز عبور (پایه برای مراحل بعدی)
userSchema.methods.changedPasswordAfter = function (
  JWTTimestamp: number,
): boolean {
  return false;
};

const User = model<IUser, Model<IUser, {}, IUserMethods>>("User", userSchema);
export default User;
```

### پیاده‌سازی کنترلر ورود (Login)

از آنجا که فیلد `password` در مدل با ویژگی `select: false` تعریف شده است، باید هنگام جستجوی کاربر برای ورود، درخواست واکشی این فیلد را به صورت صریح با `select('+password')` به Mongoose اعلام کنیم.

```typescript
import { Request, Response, NextFunction } from "express";
import User from "../models/userModel";
import { AppError } from "../utils/AppError";
import { signToken } from "../utils/auth";

export const login = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  const { email, password } = req.body;

  if (!email || !password) {
    return next(new AppError("Please provide email and password", 400));
  }

  // آوردن صریح پسورد همراه با شیء کاربر
  const user = await User.findOne({ email }).select("+password");

  if (
    !user ||
    !(await user.correctPassword(password, user.password as string))
  ) {
    return next(new AppError("Incorrect email or password", 401));
  }

  const token = signToken(user._id.toString());

  res.status(200).json({
    status: "success",
    token,
  });
};
```

> **خلاصه فصل چهارم:** منطق اعتبارسنجی رمز عبور باید در سطح Model قرار گیرد. ترکیب صحیح شرایط عدم وجود کاربر و عدم تطابق رمز عبور در یک ساختار شرطی واحد (`if`)، ضمن تمیز کردن کد، از نشت اطلاعات امنیتی جلوگیری می‌کند.

**ادامه مسیر یادگیری:** تولید توکن تنها نیمی از مسیر است. اکنون باید از این توکن برای محافظت از مسیرهای خصوصی سیستم استفاده کنیم.

---

## فصل پنجم: محافظت از مسیرها (Route Protection)

مهم‌ترین لایه امنیتی در احراز هویت توکن‌محور، ساخت میدل‌وری است که به عنوان «نگهبان» پیش از دسترسی به روت‌های محافظت‌شده اجرا شود.

### کلمه کلیدی Bearer

کلاینت برای اثبات هویت خود، باید توکن را از طریق هدر `Authorization` ارسال کند. بر اساس استاندارد جهانی HTTP، پیش از مقدار توکن باید کلمه `Bearer` به معنای «حامل» درج شود. مفهوم این کلمه در امنیت این است: «هر سیستمی که حامل این توکن باشد، اجازه ورود و دسترسی دارد».

### معماری میدل‌ور محافظت

یک میدل‌ور احراز هویت قدرتمند، ۴ مرحله بررسی قطعی انجام می‌دهد:

1. بررسی ارسال توکن در هدر `Authorization`.
2. اعتبارسنجی امضا و انقضای توکن.
3. تایید وجود کاربرِ مرتبط با توکن در دیتابیس فعلی.
4. بررسی عدم تغییر رمز عبور توسط کاربر پس از صدور توکن (مقایسه فیلد `iat` از توکن با زمان آخرین تغییر رمز).

### رفع چالش‌های تایپ‌اسکریپت (گسترش نوع Request)

در مرحله پایانی اعتبارسنجی، اطلاعات کاربرِ تاییدشده بر روی شیء `req` قرار داده می‌شود (`req.user = currentUser`) تا سایر کنترلرها نیازی به واکشی مجدد از دیتابیس نداشته باشند (مفهوم State-Passing).
از آنجایی که شیء `Request` در Express به طور پیش‌فرض فاقد ویژگی `user` است، توسعه‌دهندگان در TypeScript با خطای `Property 'user' does not exist on type 'Request'` مواجه می‌شوند.

**راه‌حل:** گسترش رابط `Request` از طریق Declaration Merging.
ابتدا فایلی به نام `express.d.ts` در مسیر `src/types/` ایجاد کنید:

```typescript
import { IUser } from "../models/userModel";

declare global {
  namespace Express {
    interface Request {
      user?: IUser;
    }
  }
}
```

سپس در فایل `tsconfig.json` پیکربندی خواندن انواع را مشخص نمایید:

```json
{
  "compilerOptions": {
    "typeRoots": ["./node_modules/@types", "./src/types"]
  },
  "include": ["src/**/*"]
}
```

### پیاده‌سازی نهایی میدل‌ور Protect

```typescript
import { Request, Response, NextFunction } from "express";
import { promisify } from "util";
import jwt, { JwtPayload } from "jsonwebtoken";
import User from "../models/userModel";
import { AppError } from "../utils/AppError";
import { config } from "../config/env";

export const protect = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  // ۱) بررسی حضور توکن
  let token;
  if (
    req.headers.authorization &&
    req.headers.authorization.startsWith("Bearer")
  ) {
    token = req.headers.authorization.split(" ")[1];
  }

  if (!token) {
    return next(
      new AppError("You are not logged in! Please log in to get access.", 401),
    );
  }

  // ۲) تایید اعتبار توکن
  const decoded = (await (promisify(jwt.verify) as any)(
    token,
    config.jwtSecret,
  )) as JwtPayload;

  // ۳) تایید وجود کاربر در دیتابیس
  const currentUser = await User.findById(decoded.id);

  if (!currentUser) {
    return next(
      new AppError(
        "The user belonging to this token does no longer exist.",
        401,
      ),
    );
  }

  // ۴) بررسی تغییر رمز عبور پس از صدور توکن
  if (currentUser.changedPasswordAfter(decoded.iat as number)) {
    return next(
      new AppError("User recently changed password! Please log in again.", 401),
    );
  }

  // قرار دادن کاربر تاییدشده در شیء Request جهت استفاده در کنترلرهای بعدی
  req.user = currentUser;

  next();
};
```

> **نکته:** در محیط Postman، تنظیمات تب Authorization روی `Bearer Token` به طور خودکار کلمه `Bearer` را قبل از توکن تزریق می‌کند. کپی کردن کلمه `Bearer` به همراه توکن در این کادر باعث ایجاد خطای `jwt malformed` خواهد شد.

> **خلاصه فصل پنجم:** میدل‌ور `protect` با عبور دادن درخواست از چهار لایه اعتبارسنجی دقیق، امنیت سیستم را تضمین کرده و با قرار دادن اطلاعات کاربر درون درخواست، مسیر دسترسی آسان به داده‌ها را برای سایر کنترلرهای سیستم باز می‌کند. استفاده از Declaration Merging برای تطبیق تایپ‌اسکریپت در این فرآیند الزامی است.

## فصل ششم: مدیریت انقضای توکن در صورت تغییر رمز عبور

یکی از آسیب‌پذیری‌های رایج در سیستم‌های مبتنی بر JWT، اعتبار داشتن توکن‌های قدیمی پس از تغییر رمز عبور توسط کاربر است. برای رفع این مشکل، باید تاریخ صدور توکن (فیلد `iat`) را با تاریخ آخرین تغییر رمز عبور مقایسه کنیم.

### به‌روزرسانی مدل داده (Model)

ابتدا باید فیلد جدیدی به نام `passwordChangedAt` از نوع `Date` به Schema و Interface کاربر اضافه شود تا زمان دقیق تغییر رمز عبور در پایگاه داده ثبت گردد.

### پیاده‌سازی متد اعتبارسنجی تغییر رمز

این منطق به عنوان یک Instance Method به نام `changedPasswordAfter` به `userSchema` اضافه می‌شود.

**چالش فنی:** زمان ذخیره‌شده در توکن (JWT Timestamp) بر حسب ثانیه محاسبه می‌شود، در حالی که شیء `Date` در جاوااسکریپت بر حسب میلی‌ثانیه کار می‌کند. برای مقایسه صحیح، باید زمان جاوااسکریپت با تقسیم بر ۱۰۰۰ به ثانیه تبدیل شود.

```typescript
userSchema.methods.changedPasswordAfter = function (
  JWTTimestamp: number,
): boolean {
  if (this.passwordChangedAt) {
    const changedTimestamp = parseInt(
      (this.passwordChangedAt.getTime() / 1000).toString(),
      10,
    );

    // بازگشت مقدار True به معنای سوخته بودن توکن است
    return JWTTimestamp < changedTimestamp;
  }

  // مقدار False یعنی رمز عبور از زمان صدور توکن تا کنون تغییر نکرده است
  return false;
};
```

## فصل هفتم: سطوح دسترسی و احراز مجوز (Authorization)

پس از احراز هویت (Authentication) که تنها ورود موفقیت‌آمیز کاربر را بررسی می‌کند، نیازمند لایه‌ای برای اعطای دسترسی (Authorization) هستیم. در این لایه بررسی می‌شود که آیا کاربرِ واردشده، صلاحیت و مجوز لازم برای انجام یک عملیات خاص (مانند حذف داده‌ها یا مشاهده لیست کاربران) را دارد یا خیر.

### تفاوت ارورهای ۴۰۱ و ۴۰۳

- **ارور ۴۰۱ (Unauthorized):** به معنای «شما وارد سیستم نشده‌اید» (نقص در احراز هویت).
- **ارور ۴۰۳ (Forbidden):** به معنای «شما وارد شده‌اید، اما اجازه دسترسی به این منبع را ندارید» (نقص در احراز مجوز).

### توسعه مدل کاربر برای پشتیبانی از نقش‌ها

ابتدا باید فیلد `role` (نقش) به مدل پایگاه داده اضافه شود. استفاده از محدودیت `enum` در Mongoose تضمین می‌کند که این فیلد تنها مقادیر از پیش‌تعریف‌شده را دریافت کند.

```typescript
const userSchema = new Schema<IUser, IUserMethods Model<IUser, {},>, IUserMethods>(
  {
    // ... سایر فیلدها
    role: {
      type: String,
      enum: ["user", "admin", "guide"],
      default: "user", // پیش‌فرض برای ثبت‌نام‌های جدید
    },
  }
);
```

### پیاده‌سازی میدل‌ور restrictTo با استفاده از Closure

در فریم‌ورک Express، توابع میدل‌ور تنها سه پارامتر ثابت (`req, res, next`) را می‌پذیرند. برای ارسال پارامترهای سفارشی (مانند لیست نقش‌های مجاز) به یک میدل‌ور، از الگوی توابع تودرتو (Closure) در جاوااسکریپت استفاده می‌شود. تابع بیرونی نقش‌ها را دریافت کرده و تابع درونی (میدل‌ور اصلی) را برمی‌گرداند.

```typescript
export const restrictTo = (...roles: string[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user?.role as string)) {
      return next(
        new AppError("You do not have permission to perform this action", 403),
      );
    }

    next();
  };
};
```

نکته: این Middleware باید حتماً بعد از protect اجرا شود، زیرا protect وظیفه‌ی احراز هویت کاربر و مقداردهی req.user را بر عهده دارد.

### نحوه محافظت از مسیرها

برای اعمال این محدودیت، کافی است توابع `protect` و `restrictTo` را به صورت زنجیره‌ای قبل از کنترلر نهایی در سیستم روتر قرار دهیم.

```typescript
// Admin only
router.delete("/:id", protect, restrictTo("admin"), deleteUser);
```

نکته معماری: ترتیب اجرای میدل‌ورها در Express اهمیت زیادی دارد. میدل‌ور restrictTo برای بررسی مجوزهای کاربر به req.user وابسته است. از آنجا که req.user در میدل‌ور protect پس از احراز هویت مقداردهی می‌شود، restrictTo باید همیشه بعد از protect قرار گیرد.

> **خلاصه فصل هفتم:** با استفاده از مفهوم Closure در جاوااسکریپت، یک میدل‌ور پویای مدیریت دسترسی ساختیم که بر اساس نقشِ استخراج‌شده از پایگاه داده، از مسیرهای حساس سیستم در برابر کاربران فاقد صلاحیت محافظت می‌کند.

## فصل هشتم: جریان بازیابی رمز عبور (Forgot / Reset Password)

جریان بازیابی رمز عبور زمانی استفاده می‌شود که کاربر رمز خود را فراموش کرده و نمی‌تواند وارد سیستم شود. به دلیل اینکه کاربر هنوز احراز هویت نشده است، نمی‌توانیم مستقیماً اجازه تغییر رمز را به او بدهیم. در این شرایط، باید یک مکانیزم امن و یک‌بارمصرف (One-Time Token) خارج از سیستم (از طریق ایمیل) برای تایید هویت او ایجاد کنیم.

### گام اول: توسعه مدل پایگاه داده

ابتدا باید زیرساخت پایگاه داده را برای پشتیبانی از فرآیند بازیابی رمز عبور آماده کنیم.

1. **افزودن فیلدهای جدید:** دو فیلد جدید به نام‌های `passwordResetToken` (برای ذخیره توکن) و `passwordResetExpires` (برای ذخیره زمان انقضا) به مدل و اینترفیس کاربر اضافه می‌شوند.
2. **تولید توکن تصادفی:** یک متد جدید (Instance Method) در مدل ایجاد می‌کنیم تا وظیفه تولید این توکن را بر عهده بگیرد. برای امنیت بیشتر، از ماژول `crypto` (تعبیه‌شده در هسته Node.js) استفاده می‌شود.

> **نکته امنیتی بسیار مهم:** توکنِ خام و ساده (Plain) هرگز نباید در پایگاه داده ذخیره شود. ما نسخه خام را به ایمیل کاربر می‌فرستیم و نسخه **هش‌شده (Hashed)** را در دیتابیس ذخیره می‌کنیم تا در صورت نشت اطلاعات سرور، توکن‌ها قابل استفاده نباشند.

```typescript
import crypto from "node:crypto";

userSchema.methods.createPasswordResetToken = function (): string {
  // ۱) تولید یک توکن تصادفی ۳۲ بایتی خام
  const resetToken = crypto.randomBytes(32).toString("hex");

  // ۲) هش کردن توکن با الگوریتم sha256 برای ذخیره امن در دیتابیس
  this.passwordResetToken = crypto
    .createHash("sha256")
    .update(resetToken)
    .digest("hex");

  // ۳) تعیین انقضا برای ۱۰ دقیقه بعد (محاسبه بر حسب میلی‌ثانیه)
  this.passwordResetExpires = new Date(Date.now() + 10 * 60 * 1000);

  // ۴) برگرداندن نسخه خام توکن جهت ارسال در بدنه ایمیل
  return resetToken;
};
```

### گام دوم: کنترلر درخواست بازیابی (Forgot Password)

این کنترلر ایمیل کاربر را دریافت کرده و در صورت وجود آن در سیستم، فرآیند تولید توکن و ارسال ایمیل را آغاز می‌کند.

1. **جستجوی کاربر:** کاربر بر اساس ایمیل ارسال‌شده در بدنه درخواست (`req.body.email`) پیدا می‌شود.
2. **تولید و ذخیره توکن:** متد ایجادشده در گام اول فراخوانی شده و داده‌ها بدون اعمال اعتبارسنجی‌های اجباری سایر فیلدها (`validateBeforeSave: false`) در دیتابیس ذخیره می‌شوند.
3. **ارسال ایمیل:** لینک حاوی توکن خام تولید شده و توسط سرویس ایمیل (مانند Nodemailer) برای کاربر ارسال می‌گردد. در صورت شکست در ارسال ایمیل، توکن‌ها از پایگاه داده پاک‌سازی می‌شوند تا از بروز خطاهای احتمالی در آینده جلوگیری شود.

```typescript
export const forgotPassword = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  const user = await User.findOne({ email: req.body.email });

  if (!user) {
    return next(new AppError("There is no user with email address.", 404));
  }

  const resetToken = user.createPasswordResetToken();
  await user.save({ validateBeforeSave: false });

  // ساخت آدرس بازگشتی (URL) شامل توکن خام
  const resetURL = `${req.protocol}://${req.get(
    "host",
  )}/api/v2/users/resetPassword/${resetToken}`;

  const message = `Forgot your password? Submit a PATCH request with your new password to: ${resetURL}`;

  try {
    await sendEmail({
      email: user.email,
      subject: "Your password reset token (valid for 10 min)",
      message,
    });

    res
      .status(200)
      .json({ status: "success", message: "Token sent to email!" });
  } catch (err) {
    // پاک‌سازی دیتابیس در صورت شکست در ارسال ایمیل
    user.passwordResetToken = undefined;
    user.passwordResetExpires = undefined;
    await user.save({ validateBeforeSave: false });

    return next(new AppError("Error sending the email. Try again later!", 500));
  }
};
```

### گام سوم: کنترلر تنظیم رمز جدید (Reset Password)

در این مرحله، کاربر روی لینک ارسال‌شده به ایمیل کلیک کرده است. توکن خام از طریق پارامترهای مسیر (`req.params.token`) دریافت می‌شود. این مسیر در روتر به صورت `router.patch("/resetPassword/:token", resetPassword);` تعریف می‌گردد.

کنترلر مربوطه چهار وظیفه اصلی دارد:

1. **رمزگشایی:** هش کردن توکنی که از URL دریافت شده تا با فرمت ذخیره‌شده در پایگاه داده تطابق پیدا کند. (استفاده از `as string` در تایپ‌اسکریپت برای تایید نوع داده الزامی است).
2. **جستجو و اعتبارسنجی:** جستجوی کاربری که توکن هش‌شده را دارد و زمان انقضای توکن او (`$gt`) هنوز به پایان نرسیده است.
3. **تغییر رمز:** جایگزینی رمز جدید (`req.body.password`) و پاک‌سازی فیلدهای توکن از دیتابیس (`undefined` کردن آن‌ها) برای جلوگیری از استفاده مجدد. با فراخوانی `user.save()`، میدل‌ورهای Mongoose به صورت خودکار فعال شده و رمز جدید را پیش از ذخیره، هش می‌کنند.
4. **ورود خودکار:** صدور و ارسال یک توکن JWT جدید تا کاربر مستقیماً وارد سیستم شود.

```typescript
export const resetPassword = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  const hashedToken = crypto
    .createHash("sha256")
    .update(req.params.token as string)
    .digest("hex");

  const user = await User.findOne({
    passwordResetToken: hashedToken,
    passwordResetExpires: { $gt: Date.now() },
  });

  if (!user) {
    return next(new AppError("Token is invalid or has expired", 400));
  }

  user.password = req.body.password;
  user.passwordResetToken = undefined;
  user.passwordResetExpires = undefined;
  await user.save();

  const token = signToken(user._id.toString());
  res.status(200).json({ status: "success", token });
};
```

### گام چهارم: به‌روزرسانی زمان تغییر رمز (Middleware)

زمانی که رمز عبور تغییر می‌کند، برای ابطال کردن توکن‌های JWT قدیمیِ کاربر، باید فیلد `passwordChangedAt` نیز به‌روزرسانی شود. این منطق از طریق یک میدل‌ور پیش از ذخیره (`pre('save')`) در فایل مدل مدیریت می‌گردد.

```typescript
userSchema.pre("save", function (next) {
  if (!this.isModified("password") || this.isNew) return next();

  // کسر ۱ ثانیه زمان برای جلوگیری از تداخل انقضای JWT در دیتابیس‌های کند
  this.passwordChangedAt = new Date(Date.now() - 1000);
  next();
});
```

> **خلاصه فصل هشتم:** پیاده‌سازی سیستم بازیابی رمز عبور نیازمند دقت بالا در مدیریت وضعیت‌های امنیتی است. استفاده از ماژول `crypto` برای ایجاد توکن‌های تصادفی، هش کردن آن‌ها پیش از ذخیره در دیتابیس، اعمال محدودیت زمانی ۱۰ دقیقه‌ای، و پاک‌سازی ردپای توکن‌ها پس از استفاده، مانع از سوءاستفاده‌های هکرها می‌شود.

---

## فصل نهم: تغییر رمز عبور کاربر واردشده (Update Password)

در این سناریو، کاربر داخل حساب کاربری خود است (توکن JWT معتبر دارد) و می‌خواهد رمز عبور خود را به دلایل امنیتی تغییر دهد. از آنجا که این مسیر محافظت‌شده است، هویت کاربر از قبل تایید شده و اطلاعات او در `req.user` در دسترس سیستم قرار دارد.

### مراحل پیاده‌سازی منطق تغییر رمز

1. **فراخوانی صریح رمز عبور:** از آنجایی که در مدل پایگاه داده فیلد رمز عبور را پنهان کرده بودیم (`select: false`)، هنگام جستجوی کاربر باید صراحتاً از `select('+password')` استفاده کنیم تا بتوانیم رمز فعلی را با دیتابیس مقایسه کنیم.
2. **اعتبارسنجی رمز فعلی:** با استفاده از متد `correctPassword` که پیش‌تر در مدل ساخته بودیم، بررسی می‌کنیم که آیا رمز فعلیِ واردشده توسط کاربر صحیح است یا خیر. (در تایپ‌اسکریپت برای جلوگیری از خطای نوع داده، از `as string` استفاده می‌کنیم).
3. **ذخیره‌سازی و اجرای میدل‌ورها:** رمز جدید را جایگزین کرده و **حتماً** از `await user.save()` استفاده می‌کنیم. استفاده از دستوراتی مانند `findByIdAndUpdate` در اینجا ممنوع است، زیرا مانع از اجرای میدل‌ورهای Mongoose (مانند هش کردن رمز جدید و تغییر `passwordChangedAt`) می‌شود.
4. **صدور توکن جدید:** با تغییر رمز عبور، توکن‌های قبلیِ کاربر از نظر امنیتی باطل می‌شوند؛ بنابراین یک توکن JWT جدید تولید و ارسال می‌کنیم تا کاربر از سیستم خارج نشود و نیازی به لاگین مجدد نداشته باشد.

### پیاده‌سازی کنترلر و مسیر (Route)

مسیر این کنترلر حتماً باید پس از توابع احراز هویت قرار بگیرد. در روتر به این شکل تعریف می‌شود:

```typescript
router.patch("/updateMyPassword", protect, updatePassword);
```

```typescript
export const updatePassword = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  // ۱) استخراج کاربر از دیتابیس (استفاده از _id برای مطابقت با تایپ‌اسکریپت)
  const user = await User.findById(req.user?._id).select("+password");

  if (!user) {
    return next(new AppError("User not found.", 404));
  }

  // ۲) بررسی صحت رمز عبور فعلی
  const isCorrect = await user.correctPassword(
    req.body.passwordCurrent,
    user.password as string,
  );

  if (!isCorrect) {
    return next(new AppError("Your current password is wrong.", 401));
  }

  // ۳) جایگزینی رمز عبور جدید و ذخیره آن
  user.password = req.body.password;

  // در صورت وجود فیلد تایید رمز در مدل، این خط نیز اضافه می‌شود:
  // user.passwordConfirm = req.body.passwordConfirm;

  await user.save(); // این متد تمام هوک‌های pre('save') را اجرا می‌کند

  // ۴) صدور توکن جدید و ارسال در پاسخ
  const token = signToken(user._id.toString());

  res.status(200).json({
    status: "success",
    token,
  });
};
```

> **خلاصه فصل نهم:** تغییر رمز عبور برای کاربران لاگین‌شده نیازمند بررسی دقیق رمز فعلی از طریق متدهای مدل و ذخیره‌سازی با `user.save()` است تا هوک‌های امنیتی Mongoose به درستی فعال شوند. در نهایت با صدور یک توکن JWT جدید، نشست کاربر تداوم می‌یابد.

---

## فصل دهم: مدیریت پروفایل کاربری (User Management)

پس از راه‌اندازی هسته‌ی امنیتی (ثبت‌نام، لاگین و رمز عبور)، کاربران باید بتوانند اطلاعات شخصی خود را مدیریت کنند. این مسیرها فقط برای کاربران احراز هویت‌شده (`protect`) در دسترس است.

### ۱. به‌روزرسانی اطلاعات پروفایل (Update Me)

برای تغییر اطلاعاتی مانند نام و ایمیل، هرگز نباید از مسیری که برای تغییر رمز عبور ساخته شده استفاده کرد؛ زیرا منطق هش کردن رمز نباید با آپدیتِ اطلاعاتِ متنی ترکیب شود.

- **فیلتر کردن داده‌ها:** بزرگترین خطر در این مسیر، ارسال فیلدهای غیرمجاز توسط کاربر است (مثلاً یک کاربر عادی `role: "admin"` را در درخواست خود ارسال کند!). برای جلوگیری از این مشکل، یک تابع کمکی (`filterObj`) ایجاد می‌کنیم تا درخواست کاربر را فیلتر کرده و تنها فیلدهای مجاز (نام و ایمیل) را استخراج کند.
- در این مسیر از متد `findByIdAndUpdate` استفاده می‌شود، زیرا فیلدهای متنی نیازی به اجرای هوک‌های امنیتی `pre('save')` ندارند.

```typescript
const filterObj = (obj: any, ...allowedFields: string[]) => {
  const newObj: any = {};
  Object.keys(obj).forEach((el) => {
    if (allowedFields.includes(el)) newObj[el] = obj[el];
  });
  return newObj;
};

export const updateMe = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  if (req.body?.password || req.body?.passwordConfirm) {
    return next(new AppError("This route is not for password updates.", 400));
  }

  const filteredBody = filterObj(req.body, "name", "email");

  const updatedUser = await User.findByIdAndUpdate(
    req.user?._id,
    filteredBody,
    {
      new: true,
      runValidators: true,
    },
  );

  res.status(200).json({ status: "success", data: { user: updatedUser } });
};
```

### ۲. حذف حساب کاربری (Soft Delete)

در پروژه‌های واقعی، اطلاعات کاربران هرگز به صورت کامل از پایگاه داده حذف نمی‌شود (Hard Delete). ما به دلایل آماری، حفظ تاریخچه خریدها و یکپارچگی پایگاه داده، از تکنیک **«حذف نرم»** استفاده می‌کنیم. در این تکنیک، یک فیلد بولی (Boolean) به نام `active` به مدل اضافه شده و هنگام درخواست حذف، مقدار آن برابر `false` قرار می‌گیرد.

```typescript
export const deleteMe = async (
  req: Request,
  res: Response,
  next: NextFunction,
) => {
  // فیلد active را برای کاربری که لاگین است، false می‌کنیم
  await User.findByIdAndUpdate(req.user?._id, { active: false });

  // وضعیت 204 (No Content) نشان‌دهنده موفقیت بدون ارسال دیتای اضافی است
  res.status(204).json({ status: "success", data: null });
};
```

### ۳. فیلتر کاربران حذف‌شده (Query Middleware)

پس از انجام حذف نرم، مشکلی که به وجود می‌آید این است که کاربرانِ حذف‌شده همچنان در کوئری‌های جستجو (مثل `findOne` هنگام لاگین یا `find` برای لیست کاربران) نمایش داده می‌شوند.

برای رفع این مشکل، یک **Query Middleware** سراسری در فایل مدل (`userModel.ts`) تعریف می‌کنیم. این هوک پیش از اجرای هر دستوری که با `find` شروع شود اجرا شده و کاربرانی که `active: false` هستند را از نتایج فیلتر می‌کند.

> **نکته مهم Mongoose:** در نسخه‌های مدرن Mongoose، زمانی که عملیات داخل میدل‌ور کاملاً همگام (Synchronous) است، نیازی به دریافت و فراخوانی تابع `next()` نداریم و در صورت استفاده ممکن است با خطای `next is not a function` مواجه شویم.

```typescript
import mongoose from "mongoose";

// استفاده از this: mongoose.Query برای رفع خطاهای تایپ‌اسکریپت در استفاده از RegExp
userSchema.pre(/^find/, function (this: mongoose.Query<any, any>) {
  // فقط کاربرانی را جستجو کن که وضعیت active آن‌ها برابر با false نیست
  this.find({ active: { $ne: false } });
});
```

> **خلاصه فصل دهم:** مدیریت پروفایل نیازمند تفکیک دقیقِ مسیرها است. تغییرات غیرامنیتی (نام، ایمیل) باید فیلتر شوند تا از تزریق داده‌های مخرب (مثل role) جلوگیری شود. همچنین حذف حساب کاربری از طریق تکنیک Soft Delete و پنهان‌سازی آن‌ها با یک Query Middleware مدیریت می‌شود.

---

## فصل یازدهم: مدیریت یکپارچه و ارسال امن توکن (JWT Cookies)

در پروژه‌های واقعی، به جای تکرار کدهای مربوط به تولید توکن و ارسال پاسخ در کنترلرهای مختلف (`signup`، `login` و `resetPassword`)، تمام این منطق را در یک تابع کمکی (Utility Function) به نام `createSendToken` تجمیع می‌کنیم. مهم‌ترین دستاوردِ این کار، ارسال توکن از طریق **کوکی‌های امن** به جای ارسال صرف در بدنه JSON است.

### ویژگی‌های کلیدی این پیاده‌سازی:

- **جلوگیری از حملات XSS (سرقت توکن):** با تنظیم پرچم `httpOnly: true`، مرورگر اجازه نمی‌دهد کدهای جاوااسکریپتِ سمت کلاینت به کوکی دسترسی پیدا کنند؛ در نتیجه هکرها نمی‌توانند توکن کاربر را بدزدند.
- **امنیت در محیط Production:** با بررسی متغیرهای محیطی، پرچم `secure: true` تنها در حالت پروداکشن فعال می‌شود تا مطمئن شویم کوکی‌ها فقط بر بستر امن HTTPS منتقل می‌شوند.
- **پاک‌سازی داده‌ها (Data Sanitization):** پیش از ارسال اطلاعات کاربر به کلاینت، فیلد رمز عبور به صورت موقت `undefined` می‌شود تا هشِ پسورد به هیچ وجه در شبکه یا مرورگر کاربر افشا نگردد.
- **استفاده از تایپ‌های Express:** با استفاده از اینترفیس `CookieOptions`، ساختار تایپ‌اسکریپت برای تنظیمات کوکی به صورت استاندارد و بدون خطا پیاده‌سازی می‌شود.

```typescript
import { Response, CookieOptions } from "express";
import { signToken } from "./signToken";
import { IUser } from "../models/userModel";
import { config } from "../config/env";

export const createSendToken = (
  user: IUser,
  statusCode: number,
  res: Response,
) => {
  // 1) Convert Mongoose ObjectId to string
  const token = signToken(user._id.toString());

  // 2) Configure standard Cookie options
  const cookieOptions: CookieOptions = {
    expires: new Date(
      Date.now() + Number(config.jwtCookieExpiresIn) * 24 * 60 * 60 * 1000,
    ),
    httpOnly: true,
  };

  // 3) Enable secure flag only in production environment (HTTPS only)
  if (config.nodeEnv === "production") cookieOptions.secure = true;

  // 4) Attach token to cookie
  res.cookie("jwt", token, cookieOptions);

  // 5) Remove password from output for security reasons
  user.password = undefined;

  // 6) Send final response
  res.status(statusCode).json({
    status: "success",
    token,
    data: {
      user,
    },
  });
};
```

> **خلاصه فصل یازدهم:** با ایجاد تابع سراسری `createSendToken`، کدهای تکراریِ مربوط به احراز هویت یکپارچه شدند. استفاده از کوکی‌های `httpOnly` در کنار مخفی‌سازی پسورد پیش از ارسال پاسخ، معماری امنیتی اپلیکیشن را کاملاً با استانداردهای پروژه‌های تجاری (Production-Ready) همگام کرد.

---

## فصل دوازدهم: امنیت لایه شبکه و محدودسازی درخواست‌ها (Rate Limiting)

پس از راه‌اندازی سیستم احراز هویت، محافظت از API در برابر حملات مخرب (مانند حملات حدس زدن رمز عبور یا Brute Force و حملات DOS برای از کار انداختن سرور) ضروری است. در این فصل، سه لایه امنیتی مهم در سطح برنامه (Application Level) به فایل اصلی سرور (`app.ts`) اضافه شده است.

### ویژگی‌های امنیتی پیاده‌سازی شده:

۱. **محدودکننده درخواست (Rate Limiter):**
با استفاده از پکیج `express-rate-limit`، تعداد درخواست‌های مجاز از یک IP مشخص محدود می‌شود (در این پروژه: ۱۰۰ درخواست در هر ۱۵ دقیقه). اگر کاربری از این حد عبور کند، سرور درخواست‌های بعدی او را با خطای مناسب مسدود می‌کند.

۲. **اعتماد به پراکسی (Trust Proxy):**
در محیط‌های پروداکشن (ابری یا داکر)، درخواست‌ها پیش از رسیدن به Node.js از یک سرور واسط (مانند Nginx یا Load Balancer) عبور می‌کنند. با فعال کردن `app.set("trust proxy", 1)`، به اکسپرس اجازه می‌دهیم تا آی‌پیِ واقعی کاربر را از هدر درخواست بخواند؛ در غیر این صورت، Rate Limiter آی‌پیِ سرورِ واسط را مسدود کرده و کل سیستم از کار می‌افتد.

۳. **محدودیت حجم بدنه درخواست (Payload Size Limit):**
برای جلوگیری از حملات بارگذاری سنگین (ارسال داده‌های حجیم برای پر کردن حافظه سرور)، متد پردازشگرِ اکسپرس محدود شده است (`limit: "10kb"`). با این کار، سرور از پردازش درخواست‌هایی با بادیِ بزرگتر از ۱۰ کیلوبایت امتناع می‌کند.

```typescript
import express, { Express } from "express";
import userRouter from "./routes/user.routes";
import { config } from "./config/env";
import morgan from "morgan";
import { AppError } from "./utils/AppError";
import { globalErrorHandler } from "./middlewares/errorHandler";
import rateLimit from "express-rate-limit";

const app: Express = express();

// 1) Trust proxy: Crucial for production to read the real client IP
app.set("trust proxy", 1);

if (config.nodeEnv === "development") {
  app.use(morgan("dev"));
}

// 2) Rate Limiting: Prevent Brute Force and DOS attacks
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes window
  limit: 100, // Limit each IP to 100 requests per window
  standardHeaders: "draft-8",
  legacyHeaders: false,
  message: "Too many requests from this IP, please try again later.",
});

// Apply rate limiter to all /api routes
app.use("/api", limiter);

// 3) Body parser with size limit to prevent payload attacks
app.use(express.json({ limit: "10kb" }));

// Mount routes
app.use("/api/v2/users", userRouter);

// Handle unhandled routes (404)
app.all("*", (req, res, next) => {
  next(new AppError(`Cannot find ${req.originalUrl} on this server!`, 404));
});

// Global Error Handler
app.use(globalErrorHandler);

export default app;
```

> **خلاصه فصل دوازدهم:** با ترکیب سه ابزارِ `Rate Limiting`، مدیریتِ `Proxy` و محدودسازیِ `Body Parser`، اپلیکیشن در برابر رایج‌ترین حملاتِ لایه‌ی شبکه ایمن‌سازی شد. این پیکربندی‌ها برای استقرار ایمن پروژه در محیط‌های واقعی (Production) الزامی هستند.

---

## فصل سیزدهم: ایمن‌سازی هدرهای HTTP (Helmet)

پروتکل HTTP دارای هدرهایی (Headers) است که همراه با هر پاسخ از سرور به کلاینت ارسال می‌شوند. هدرهای پیش‌فرض Express ممکن است اطلاعات حساسی از سرور را فاش کنند (مانند نسخه فریم‌ورک) یا در برابر حملات تزریق اسکریپت ضعیف باشند. در این فصل، با استفاده از پکیج **Helmet**، یک لایه دفاعی قدرتمند روی هدرهای برنامه قرار دادیم.

### مزایای کلیدی استفاده از Helmet:

- **مخفی کردن اطلاعات سرور:** هدرِ `X-Powered-By` (که به هکرها می‌گوید سرور شما با اکسپرس نوشته شده) را حذف می‌کند.
- **جلوگیری از حملات XSS:** هدرِ `X-XSS-Protection` را فعال می‌کند تا مرورگرها کدهای مشکوک جاوااسکریپت را متوقف کنند.
- **جلوگیری از Clickjacking:** با تنظیم `X-Frame-Options`، اجازه نمی‌دهد سایت‌های دیگر وب‌سایتِ شما را داخل یک `iframe` باز کنند (جلوگیری از کلیک‌های فریبنده).
- **اجبار به ارتباط امن (HSTS):** با تنظیم `Strict-Transport-Security`، مرورگر را مجبور می‌کند تا همیشه از HTTPS استفاده کند.

پکیج Helmet باید همیشه در ابتدای فایل اصلی سرور (بالاتر از سایر میدل‌ورها) فراخوانی شود تا امنیت آن بر تمام مسیرهای سامانه اِعمال گردد.

```typescript
import express, { Express } from "express";
import helmet from "helmet";
import userRouter from "./routes/user.routes";
import { config } from "./config/env";
import morgan from "morgan";
import { AppError } from "./utils/AppError";
import { globalErrorHandler } from "./middlewares/errorHandler";
import rateLimit from "express-rate-limit";

const app: Express = express();

// 1) Set Security HTTP Headers: Must be at the top to secure all subsequent middlewares
app.use(helmet());

// 2) Trust proxy for production environments
app.set("trust proxy", 1);

// 3) Development Logging
if (config.nodeEnv === "development") {
  app.use(morgan("dev"));
}

// 4) Rate Limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 100,
  standardHeaders: "draft-8",
  legacyHeaders: false,
  message: "Too many requests from this IP, please try again later.",
});
app.use("/api", limiter);

// 5) Body Parser
app.use(express.json({ limit: "10kb" }));

// 6) Routes
app.use("/api/v2/users", userRouter);

// 7) Handle Unhandled Routes
app.all("*", (req, res, next) => {
  next(new AppError(`Cannot find ${req.originalUrl} on this server!`, 404));
});

// 8) Global Error Handler
app.use(globalErrorHandler);

export default app;
```

> **خلاصه فصل سیزدهم:** با اضافه کردن Helmet، چهارده لایه‌ی امنیتیِ پنهان به صورت خودکار بر روی هدرهای HTTP تنظیم شد. این تغییرِ سریع و یک‌خطی، آسیب‌پذیری‌های رایج تحت وب (از جمله Clickjacking و XSS) را در لایه مرورگر مسدود می‌کند.

---

## فصل چهاردهم: پاک‌سازی داده‌ها و جلوگیری از تزریق کد (Data Sanitization)

با وجود محدود کردن درخواست‌ها (Rate Limiting) و ایمن‌سازی هدرها (Helmet)، سرور همچنان در برابر داده‌های مخربی که در بدنه (Body) درخواست ارسال می‌شوند آسیب‌پذیر است. هکرها می‌توانند کدهایی را ارسال کنند که مستقیماً پایگاه داده را هدف قرار می‌دهد یا در مرورگر سایر کاربران اجرا می‌شود. برای رفع این خطرات، از دو میدل‌ور مهم پس از `express.json` استفاده می‌کنیم:

این بخش یکی از جذاب‌ترین مفاهیم امنیتی بک‌اند یعنی **«پاک‌سازی داده‌ها» (Data Sanitization)** است. هرگز نباید به داده‌ای که از سمت کاربر می‌آید اعتماد کرد، حتی اگر کاربر لاگین کرده باشد!

### ۱. جلوگیری از تزریق NoSQL (NoSQL Query Injection)

در پایگاه داده MongoDB، هکرها می‌توانند با ارسال عملگرهایی مانند `$gt` (بزرگتر از) به جای رمز عبور، سیستم احراز هویت را فریب دهند (مثلاً ایمیل را می‌دانند و برای رمز عبور می‌گویند: رمزی که طول آن بیشتر از صفر است!).
پکیج `express-mongo-sanitize` تمام درخواست‌های ورودی (`req.body`، `req.query` و `req.params`) را بررسی کرده و هرگونه کلیدی که با علامت دلار (`$`) یا نقطه (`.`) شروع شود را حذف می‌کند.

### ۲. جلوگیری از حملات XSS (Cross-Site Scripting)

هکرها ممکن است در فیلدهای متنی (مثل نام کاربر)، کدهای مخرب HTML یا جاوااسکریپت وارد کنند تا این کدها در مرورگرِ قربانیانِ دیگر اجرا شود. با توجه به منسوخ شدن پکیج قدیمی `xss-clean`، از جایگزین استاندارد آن یعنی `express-xss-sanitizer` استفاده کردیم. این پکیج کدهای مخرب را به رشته‌های متنیِ بی‌خطر تبدیل می‌کند (مثلاً تگِ `<script>` را غیرفعال می‌کند).

```typescript
import express, { Express } from "express";
import userRouter from "./routes/user.routes";
import { config } from "./config/env";
import morgan from "morgan";
import { AppError } from "./utils/AppError";
import { globalErrorHandler } from "./middlewares/errorHandler";
import rateLimit from "express-rate-limit";
import helmet from "helmet";
import mongoSanitize from "express-mongo-sanitize";
import { xss } from "express-xss-sanitizer";

const app: Express = express();

// 1) Set security HTTP headers
app.use(helmet());

// 2) Trust proxy for production environment
app.set("trust proxy", 1);

// 3) Development logging
if (config.nodeEnv === "development") {
  app.use(morgan("dev"));
}

// 4) Rate Limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 100,
  standardHeaders: "draft-8",
  legacyHeaders: false,
  message: "Too many requests from this IP, please try again later.",
});
app.use("/api", limiter);

// 5) Body parser, reading data from body into req.body
app.use(express.json({ limit: "10kb" }));

// 6) Data sanitization against NoSQL query injection
app.use(mongoSanitize());

// 7) Data sanitization against XSS (Cross-Site Scripting)
app.use(xss());

// 8) Routes
app.use("/api/v2/users", userRouter);

// 9) Handle Unhandled Routes
app.all("*", (req, res, next) => {
  next(new AppError(`Cannot find ${req.originalUrl} on this server!`, 404));
});

// 10) Global Error Handler
app.use(globalErrorHandler);

export default app;
```

> **خلاصه فصل چهاردهم:** یک اپلیکیشن امن باید ورودی‌های خود را ضدعفونی کند! استفاده ترکیبی از `mongoSanitize` و `xss` تضمین می‌کند که داده‌های دریافتی از کلاینت، نه برای پایگاه داده و نه برای مرورگر دیگر کاربران، هیچ‌گونه تهدیدی به همراه نخواهند داشت. رعایت ترتیب این میدل‌ورها (دقیقاً پس از پردازشگرِ Body) بسیار حیاتی است.

---

## فصل پانزدهم: جلوگیری از آلودگی پارامترها (HTTP Parameter Pollution)

یکی از حملات زیرکانه به سرورهای Node.js، ارسال چندین پارامتر هم‌نام در URL است (مثلاً `?sort=price&sort=duration`). در این حالت، فریم‌ورک Express آن پارامتر را از یک رشته (String) به یک آرایه (Array) تبدیل می‌کند. اگر در کدهای بک‌اند انتظار دریافت یک رشته را داشته باشیم و از متدهایی مانند `split` یا `toUpperCase` استفاده کنیم، سرور با خطا مواجه شده و از کار می‌افتد (Crash).

برای جلوگیری از این مشکل، از پکیج `hpp` استفاده می‌کنیم. این میدل‌ور تمام پارامترهای تکراری را حذف کرده و تنها آخرین مقدار ارسال‌شده را نگه می‌دارد.

**قابلیت Whitelist (لیست سفید):**
در مواردی که واقعاً نیاز داریم کاربر بتواند یک پارامتر را چند بار ارسال کند (مانند فیلتر کردن قیمت‌های مختلف در جستجو)، نام آن پارامترها را در آرایه `whitelist` قرار می‌دهیم تا `hpp` آن‌ها را نادیده بگیرد.

```typescript
import express, { Express } from "express";
import userRouter from "./routes/user.routes";
import { config } from "./config/env";
import morgan from "morgan";
import { AppError } from "./utils/AppError";
import { globalErrorHandler } from "./middlewares/errorHandler";
import rateLimit from "express-rate-limit";
import helmet from "helmet";
import mongoSanitize from "express-mongo-sanitize";
import { xss } from "express-xss-sanitizer";
import hpp from "hpp";

const app: Express = express();

// 1) Set security HTTP headers
app.use(helmet());

// 2) Trust proxy for production environment
app.set("trust proxy", 1);

// 3) Development logging
if (config.nodeEnv === "development") {
  app.use(morgan("dev"));
}

// 4) Rate Limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 100,
  standardHeaders: "draft-8",
  legacyHeaders: false,
  message: "Too many requests from this IP, please try again later.",
});
app.use("/api", limiter);

// 5) Body parser, reading data from body into req.body
app.use(express.json({ limit: "10kb" }));

// 6) Data sanitization against NoSQL query injection
app.use(mongoSanitize());

// 7) Data sanitization against XSS
app.use(xss());

// 8) Prevent parameter pollution (MUST be before routes)
app.use(
  hpp({
    whitelist: [
      "duration",
      "ratingsQuantity",
      "ratingsAverage",
      "maxGroupSize",
      "difficulty",
      "price",
    ],
  }),
);

// 9) Routes
app.use("/api/v2/users", userRouter);

// 10) Handle Unhandled Routes
app.all("*", (req, res, next) => {
  next(new AppError(`Cannot find ${req.originalUrl} on this server!`, 404));
});

// 11) Global Error Handler
app.use(globalErrorHandler);

export default app;
```
