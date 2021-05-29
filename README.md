# hybrid_sort_homework

Для проведения измерений был написан приведенный ниже код.
Время сортировки каждого алгоритма было измерено 10000 раз и усреднено.

```c++
#include <iostream>
#include <vector>
#include <cassert>
#include <cmath>
#include <functional>
#include <algorithm>
#include <ctime>
#include <chrono>
#include <fstream>

using namespace std;


#define MIN_DATA_SIZE               16
#define MAX_DATA_SIZE               1'000'000
#define MAX_DATA_VALUE              1'000'000
#define PROGRESSION_COEF            2
#define SAME_DATA_SIZE_SORTS_COUNT  10000


std::vector<int> generate_data(size_t a_data_size)
{
    std::vector<int> data;
    data.reserve(a_data_size);
    for (size_t i = 0; i < a_data_size; ++i) {
        data.push_back(rand() % MAX_DATA_VALUE);
    }
    return data;
}


void sorts_compare(std::vector<
    std::function<void(vector<int>&, size_t, size_t)>> a_sort_functions)
{
    ofstream sorts_results_file("sorts_results.csv", ios::trunc);
    sorts_results_file << "data_size,insertion_sort,bubble_sort,merge_sort,quick_sort, hybrid_sort" << endl;


    for (size_t ds = MIN_DATA_SIZE; ds < MAX_DATA_SIZE; ds *= PROGRESSION_COEF) {
        cout << "data size sort now: " << ds << endl;
        sorts_results_file << ds << ",";

        for (size_t i = 0; i < a_sort_functions.size(); ++i) {
            double average_sort_time = 0;
            for (size_t j = 0; j < SAME_DATA_SIZE_SORTS_COUNT; ++j) {
                std::vector<int> data_to_sort(generate_data(ds));

                auto sort_start_time = std::chrono::steady_clock::now();

                a_sort_functions[i](data_to_sort, 0, data_to_sort.size());

                auto sort_end_time = std::chrono::steady_clock::now();

                std::chrono::duration<double> sort_time =
                    (sort_end_time - sort_start_time) * 1000;

                average_sort_time += sort_time.count();
            }

            average_sort_time /= SAME_DATA_SIZE_SORTS_COUNT;
            sorts_results_file << average_sort_time << ",";
        }
        sorts_results_file << endl;
    }
}


int main()
{
    std::srand(std::time(0));

    sorts_compare({ insertion_sort, bubble_sort, merge_sort, quick_sort, hybrid_sort });

    return 0;
}
```

Для начала определимся с сортировкой на маленьких массивах данных.
Измерим время алгоритмов сортировки с высокой дискретностью по размеру данных.
Результаты (все графики построены в логарифмической шкале):
https://datawrapper.dwcdn.net/RsiVI/2/
https://datawrapper.dwcdn.net/ARfUy/2/

Из графиков можно увидеть, что, как минимум, до размера данных 150 сортировка вставкой превосходит все остальные.
Т.к. до размера данных, меньших 4, сортировки пузырьком и вставкой примерно совпадают по времени исполнения, сортировку пузырьком можно исключить, чтобы не усложнять алгоритм.

Снимем результаты с менее высокой дискретностью для бОльших размеров данных, исключив сортировку пузырьком, для ускорения измерений.
Результаты: https://datawrapper.dwcdn.net/6w8mK/2/

Из графиков можно увидеть, что примерно после размера данных 370 сортировка вставкой начинает уступать быстрой сортировке.

Чтобы понять, превзойдет ли сортировка слиянием быструю сортировку, исключим сортировку вставкой для ускорения измерений.
Результаты: https://datawrapper.dwcdn.net/cFcbD/4/

Из графиков можно увидеть, что, как минимум, для размеров данных меньше 262000, быстрая сортировка превосходит сортировку слиянием, поэтому исключим сортировку слиянием из алгоритма.

Т.о. алгоритм будет состоять из двух сортировок: сортировка вставкой и быстрая сортировка.
Сортировка вставкой будет применяться для массивов, размер которых не превышает 370. Для всех остальных массивов будет применяться быстрая сортировка

Ниже приведен код финального алгоритма:

```c++
#include <iostream>
#include <vector>
#include <cassert>
#include <cmath>
#include <functional>
#include <algorithm>
#include <ctime>
#include <chrono>
#include <fstream>

using namespace std;


ostream& operator<<(ostream& a_str, vector<int> a_vec)
{
    a_str << "{ ";
    for (size_t i = 0; i < a_vec.size(); ++i) {
        a_str << a_vec[i];
        if (i < a_vec.size() - 1) {
            a_str << ", ";
        }
    }
    a_str << " }";
    return a_str;
}


void sort_assert(const vector<int>& a, const vector<int>& b)
{
    if (a != b) {
        cout << a << " != " << b << endl;
        assert(false);
    }
}


void random_test(std::vector<std::function<void(vector<int>&, size_t, size_t)>> a_sort_foos)
{
    for (size_t i = 0; i < 1000; ++i) {

        size_t vector_size = 1 + (std::rand() % 1000);
        std::vector<int> data(vector_size);
        for (size_t i = 0; i < data.size(); ++i) {
            data[i] = std::rand() % 1000;
        }

        std::vector<int> sorted_data(data);
        sort(sorted_data.begin(), sorted_data.end());

        for (size_t i = 0; i < a_sort_foos.size(); ++i) {
            std::vector<int> data_copy = data;
            a_sort_foos[i](data_copy, 0, data_copy.size());
            sort_assert(data_copy, sorted_data);
        }
    }
    cout << "random_test passed" << endl;
}


void insertion_sort(std::vector<int>& a_arr, size_t p, size_t q)
{
    for (size_t i = p + 1; i < q; ++i) {
        for (size_t j = i; j > 0; --j) {
            if (a_arr[j] < a_arr[j - 1]) {
                swap(a_arr[j], a_arr[j - 1]);
            } else {
                break;
            }
        }
    }
}


size_t partition(vector<int>& a_arr, size_t p, size_t q)
{
    size_t split_idx = p + (rand() % (q - p));
    int split_value = a_arr[split_idx];
    swap(a_arr[split_idx], a_arr[q - 1]);

    size_t i = p;
    for (size_t j = p; j < q - 1; ++j) {
        if (a_arr[j] <= split_value) {
            swap(a_arr[j], a_arr[i]);
            i += 1;
        }
    }
    swap(a_arr[i], a_arr[q - 1]);
    return i;
}


void hybrid_sort(std::vector<int>& a_arr, size_t p, size_t q)
{
    size_t data_size = q - p;
    if (data_size <= 370) {
        insertion_sort(a_arr, p, q);
    } else {
        size_t split_idx = partition(a_arr, p, q);
        hybrid_sort(a_arr, p, split_idx);
        hybrid_sort(a_arr, split_idx, q);
    }
}


int main()
{
    std::srand(std::time(0));

    random_test({ hybrid_sort });

    return 0;
}
```

Ради интереса, сравним время исполнения финального алгоритма с остальными.
Результаты: https://datawrapper.dwcdn.net/cAq3R/1/

Из графиков можно сделать вывод, что hybrid_sort достиг цели, ради которой он создавался.
