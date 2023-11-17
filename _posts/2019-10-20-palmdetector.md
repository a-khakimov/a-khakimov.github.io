---
layout: post
title: 'PalmDetector : Приложи ладонь к сканеру'
date: 2019-10-20 00:01:30 +0000
categories: [Решение задач]
tags: [c/c++, qt, qtcarts, algorythms, gsl, practice]
---

Как то мне в руки попало достаточно интересное тестовое задание. Академический интерес взял верх и я решил посидеть над этой задачкой. Мое решение не претендует на оптимальность и правильность. Мне просто интересно было ее решить.

### Исходные данные

Суть задания заключается в следующем - написать программу, которая по изображению снимков со сканера вен ладоней определяет приложена ли ладонь к сканеру.
Исходные данные - несколько снимков с заранее известным результатом. Нужно скормить их программе, а программа, в свою очередь, должна сказать - приложена ладонь или нет.
![](/assets/img/palmdetector/imgslist.png){: .dark .w-75 .shadow .rounded-10 w='600' }

### Результат

Программа с графическим интерфейсом с возможностью выбора изображения из списка. После выбора изображение анализируется и после анализа выдается результат в виде надписи **Good** или **Bad**.

![](/assets/img/palmdetector/palmdetector_good.png){: .dark .w-75 .shadow .rounded-10 w='600' }

![](/assets/img/palmdetector/palmdetector_bad.png){: .dark .w-75 .shadow .rounded-10 w='600' }

### Алгоритм

Алгоритм анализа изображения довольно простой. Для начала создал класс **ImageAnalyser** со следующим интерфейсом
```cpp
class ImageAnalyser
{
public:
    ImageAnalyser();
    explicit ImageAnalyser(const QImage&);
    bool analyze(const QImage&);
    bool analyze();
    std::vector<std::vector<int>> data();
    virtual ~ImageAnalyser();
};
```

Внутри этого класса решил условно разделить изображение на 4 части для каждого источника света. И для каждого изображения расчитать среднюю яркость относительно осей **Х** и **У**. Наглядно это продемонстрировано на изображении ниже.

![](/assets/img/palmdetector/plot.gif){: .dark .w-75 .shadow .rounded-10 w='500' }

В результате получим восемь графиков со средним уровнем яркости.

![](/assets/img/palmdetector/algorythm.gif){: .dark .w-75 .shadow .rounded-10 w='500' }

Далее нужно произвести анализ этих графиков. Я решил использовать функцию корреляции сравнив полученные графики с некоторым "идеальным" графиком. Идеальный график в данном случае это просто прямоугольник, который я получаю следующим способом:

```cpp
std::vector<int> ImageAnalyser::prepare_ideal_array(const std::vector<int>& array)
{
    unsigned long min = static_cast<unsigned long>(array.size() * 0);
    unsigned long max = static_cast<unsigned long>(array.size() * 0.45);
    int ideal_value = 100;

    std::vector<int> ideal;
    ideal.resize(array.size());

    for(unsigned long i = min; i < max; ++i) {
        ideal[i] = ideal_value;
    }

    return ideal;
}
```

Для сравнения графиков и, соответственно, получения значения корреляции я использовал функцию **gsl_stats_correlation**, реализацию которого честно украл из [GNU Scientific Library](https://www.gnu.org/software/gsl/doc/html/statistics.html).

```cpp
double ImageAnalyser::gsl_stats_correlation(const std::vector<int>& data)
{
    std::vector<int> ideal = prepare_ideal_array(data);
    const int stride1 = 1;
    const int stride2 = 1;

    double sum_xsq = 0.0;
    double sum_ysq = 0.0;
    double sum_cross = 0.0;

    double mean_x = data[0];
    double mean_y = ideal[0];

    for (unsigned int i = 1; i < data.size(); ++i) {
        double ratio = i / (i + 1.0);
        double delta_x = data[i * stride1] - mean_x;
        double delta_y = ideal[i * stride2] - mean_y;
        sum_xsq += delta_x * delta_x * ratio;
        sum_ysq += delta_y * delta_y * ratio;
        sum_cross += delta_x * delta_y * ratio;
        mean_x += delta_x / (i + 1.0);
        mean_y += delta_y / (i + 1.0);
    }

    double r = sum_cross / (sqrt(sum_xsq) * sqrt(sum_ysq));

    return r;
}
```

Далее нужно просто проанализировать значения корреляции. Я решил, что если хоть одно значение корреляции меньше 0,5 то ладонь к сенсору не приложена или приложена плохо.

```cpp
bool ImageAnalyser::is_good(const vector<double>& correlation, const vector<int>& maximums)
{
    bool result = true;
    double min_corr = *std::min_element(correlation.begin(), correlation.end());
    if (min_corr < 0.5) {
        result = false;
    }
    double min_val = *std::min_element(maximums.begin(), maximums.end());
    if (min_val < 30) {
        result = false;
    }
    return result;
}
```

Так же из кода видно, что производится анализ уровня яркости - если значение меньше 30, то так же считаем, что ладонь не приложена.

### Стек используемых технологий

* C/C++
* Qt Creator
* QtCharts
* GNU Scientific Library

#### Исходники

[https://github.com/techlinked/PalmDetector.git](https://github.com/techlinked/PalmDetector.git)
