# Современные плюсцы))
## Фичи новых стандартов
### Фичи языка
- Ослабленные требования к `constexpr`, позволящие писать более сложный код. Одна из его фишек — это более строгая проверка и запрет на UB в теле функции. Константы полеезнее также делать как `constexpr`, а не дефайнами, ибо у компилятора будет  информация о типах и он сможет предупредить о неожиданных переполнениях, например
- Больше трейтов в стандартной библиотеке, позволяющих вместо своих непроверенных костылей использовать стандартные провереннные средства и оставлять код более читаемым
- `constexpr` шаблонные переменные, позволяющие вместо `typename std::enable_if<std::is_same<T1, T2>::value, U>::type` просто `std::enable_if_t<std::is_same_v<T1, T2>, U>`, что опять же увеличивает читаемость кода, освобождая его от служебных нагромождений
- Структурное связывание, позволяющее давать осмысленные имена при пробеге по мапам, вместо пары с ничего не говорящими `.first` и `.second`
- `constexpr if`, облегчающий написание шаблонных функций со SFINAE — делающий их более читабельными вместо копипасты общих участков функции в две почти идентичные (пример в simple_job.h)
- Появление delete, с указанием размера памяти, на которую ссылается данный указатель. Это облегчает работу аллокатора, особенно с популярынми строками тем, что не тригерит поиск соовтвествия размера памяти по данному указателю в хэшмапах, а сразу идёт в нужную внутреннюю структурку. (Вообще от экспертов по с++ бытует мнение, что такая молель была ошибкой. И аллокаторы должны возвращать не просто указатеть, а пару (указатель, размер)

### Фичи стандартной библиотеки
#### Это же не часть языка, можно взять откуда-нибудь!
Действительно, возможности стандартной библиотеки, как правило, реализуемы в рамках старых стандартов,
однако зависеть от буста мы всё ещё боимся, а реализация своими руками связана с рядом проблем:
- Зачастую даже для достаточно тупых обёрток (вроде `string_view`) приходится писать много бойлерплейта,
из-за чего усилий на написание уходит не то чтобы пренебрежимо мало
- Когда хочешь воспользоваться фичей, которую, как выясняется, в наших велосипедах пока не реализовали,
приходится идти и реализовывать её, выпадая из контекста, что пагубно влияет на продуктивность
- Иногда случаются баги, притом сильно чаще, чем в коде, которые постоянно используется тысячами людей
#### Что за фичи такие
- `std::chrono` — работа с временем
- Алгоритмы вроде `count_if`
#### Чем полезно их наличие / вредно отсутствие
#### Чем можно заменить при использовании старых стандартов языка
## Фичи современных компиляторов
### Продвинутые оптимизаторы
Согласно недавнему [вбросу](https://pastebin.mvk.com/JcXho1wlVw8wLEx9LAkGbJK1fuj6gaScOMNG4nzTToGL05vlpDPi4TiDZFMjDgqRAYPhhcIqWqrDDkrN.hs) в чат с вбросами, такая всеми силами заоптимизированная простыня:
```c
static inline double __vector_product(const double *x, const double *y, const int size) {
#if defined(__x86_64__)
  __v2df as, bs, result;
  double temp[2];
  temp[0] = temp[1] = 0.0;
  result = _mm_loadu_pd(temp);
  int i = 0;
  for (i = 0; i + 1 < size; i += 2) {
    as = _mm_loadu_pd(x + i);
    bs = _mm_loadu_pd(y + i);
    result += as * bs;
  }
  _mm_storeu_pd(temp, result);
  temp[0] += temp[1] + (i < size ? x[size - 1] * y[size - 1] : 0.0);
  return __fpclassify(temp[0]) == FP_SUBNORMAL ? 0.0 : temp[0];
#elif defined(__arch64__)
  float64x2_t xf, yf, result;
  double temp[2] = {0.0, 0.0};
  result = vld1q_f64(temp);

  int i;
  for (i = 0; i + 1 < size; i += 2) {
    xf = vld1q_f64(x + i);
    yf = vld1q_f64(y + i);

    result = vaddq_f64(result, vmulq_f64(xf, yf));
  }
  vst1q_f64(temp, result);
  temp[0] += temp[1] + (i < size ? x[size - 1] * y[size - 1] : 0.0);

  return __fpclassify(temp[0]) == FP_SUBNORMAL ? 0.0 : temp[0];
#else // __arch64__
  double result = 0.0;

  for (int i = 0; i < size; ++i) {
    result += x[i] * y[i];
  }
  return __fpclassify(result) == FP_SUBNORMAL ? 0.0 : result;
#endif
}
```
по производительности проигрывает (во всяком случае, на рабочем ноуте не выигрывает) такому максимально простому коду:
```c
__attribute__((optimize("-Ofast")))
double dot_product(const double *x, const double *y, const int size) {
  double result = 0;
  for (int i = 0; i < size; i++) {
    result += x[i] * y[i];
  }
  return __fpclassify(result) == FP_SUBNORMAL ? 0.0 : result;
}
```
Такой результат получается при использовании компиляторов актуальных версий; мы же пишем код, который невыразителен, непереносим (то есть для арма кусок кода пришлось переписать) и при этом неэффективен.

Есть, конечно, и другие примеры: например, [в ряде случаев](https://godbolt.org/z/G-I6_2) свежие версии gcc
справляются преобразовать switch в возврат значения по индексу из массива, а старые — нет.
### Чем мешает отсутствие возможности компилировать ими на проде
Тут @eugene536 набросит про отладочные символы, время компиляции KPHP и вот это вот всё.
