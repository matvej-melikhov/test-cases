### Задание
Сделать оценку ответов на вопросы на валидационной выборке (определяется по id).
Попробовать минимум 3 модели, выбрать лучшую, объяснить свой выбор, аргументируя его с помощью метрик качества. На выходе должны получиться `.py`/`.ipynb` файлы с моделями, `xlsx` файл с оценками (можно в формате исходника) + файл `readme` с инструкцией к запуску и комментарием.

### Постановка задачи
В датасете присутствуют 3 колонки с ответами на вопросы, которые я назвал `answer1`, `answer2` и `answer3`, а также оценки (от 0 до 3 с шагом 0.5) для каждого из ответов — `score1`, `score2`, `score3`. Сумма по трем оценкам для конкретного человека хранится в колонке `result`.

Так как строгих условий по решению задачи нет, то я выбрал два подхода к решению задачи: 
- **классификация ответов** на положительные и отрицательные (причем можно классифицировать каждый ответ по отдельности, а можно считать их как один большой ответ на несколько вопросов, и классифицировать уже его);
- **предсказание оценки (регрессия)** в диапазоне \[0; 3\] (в исходном датасете оценки принимают ограниченное число значений — обычно с шагом 0.5 — можно повторить такую особенность, используя деревянные алгоритмы, также можно научить модель делать непрерывные предсказания на диапазоне, возможно, это будет лучше отвечать задачам бизнеса).

### Разведывательный анализ
Я начал со знакомства с данными, чтобы лучше разобраться в их структуре и, возможно, выделить некоторые закономерности, которые можно использовать при обучении — файл `EDA.ipynb`.

Например, удалось узнать, что:
1) нецелые оценки (кроме 1.5) встречаются в выборке сильно реже целых — в таком случае, возможно, стоит оценивать ответы по 3 балльной шкале с целыми значениями;
2) имеется некоторая связь между длинной ответа (количество символов в ответе) и оценкой: например, что самые длинные ответы обычно получают высший балл и что начиная с некоторой длины ответа нулевой балл за ответ практически отсутствует;
3) есть связь между длиной ответов одного и того же человека, что логично: обычно человек либо отвечает на все вопросы развернуто, либо на все довольно коротко (сильного контраста нет);
4) (следует из 2 и 3): есть связь между оценками на ответы одного и того же человека: например, если человек получает плохую оценку за первые 2 ответа, то он почти наверняка получит плохую оценку и за 3-ий ответ — это также можно использовать при последовательном обучении моделей (следующая модель также может ориентироваться на оценки, которые предсказали предыдущие модели).

### Обучение моделей
В процессе решения этих задач NLP я решил двигаться от простого к сложному и начал с самых базовых моделей. Я выбрал 3 основных подхода:  
- **CatBoost**: в последних версиях библиотеки добавили поддержку текстовых фичей — файл `vanilla_catboost.ipynb`.  
- **TF-IDF + классическая модель** (KNN, LogReg, NaiveBayes) — файл `tf-idf_and_classic_model`.  
- **BERT + классическая модель** (KNN, LogReg, NaiveBayes) — файл `bert_and_classic_model`.  

В каждом из подходов я старался найти оптимальное решение как для задачи регрессии оценок, так и для задачи классификации ответов. Я считаю, что более правильным подходом к решению этой задачи было бы рассматривать её как задачу классификации, так как это упрощает её и делает более интерпретируемой как для модели, так и для бизнеса: ответ засчитан или нет. *Но это исключительно мое мнение.*

Также при анализе данных было замечено, что длина ответа некоторым образом коррелирует с оценкой за него, поэтому помимо векторных признаков текста я использовал в качестве признаков длину ответа или ответов, если обучалась одна большая модель. Это позволило немного выиграть в качестве (проверено на черновиках).

Помимо получения регрессионных оценок обычными моделями я в качестве эксперимента попробовал предсказывать вероятности отнесения к положительному классу и масштабировать до \[0; 3\]. Очевидно, что лучшие результаты показали именно регрессионные модели.

Я старался комментировать все этапы работы в приложенных ноутбуках. Использованные библиотеки, которые не входят в стандартную комплектацию языка, можно установить прямо при запуске блокнота, в ячейках оставлял код для загрузки (правда он закомментирован).

Для задачи классификации основной метрикой была выбрана `f1_macro`, так как она в равной степени учитывает точность и полноту предсказания как положительного, так и отрицательного класса. Я думаю, что она хорошо подходит для данной задачи, учитывая также, что имеется некоторый дисбаланс классов.

Для задачи регрессии использовались метрики `RMSE`, `MAE`, которые хорошо интепретируемы, а также `R-squared`. `MAE` показалось наиболее понятной для данной задачи, поэтому по ней модели сравниваются в таблице.

Для задачи классификации лучшее качество показали модели с векторизацией TF-IDF и логистической регрессией, причем обучается и используется отдельно для каждого из ответов.

Для задачи регрессии наилучшее качество показала модель с объединенными ответами и использованием CatBoostRegressor.

### Финальное предсказание
Финальное предсказание производится в файле `final_prediction.ipynb`. В файлах `class_result_dataset.xlsx` и `reg_result_dataset.xlsx` представлены исходные датасеты с доставленными предсказаниями. 

В случае классификации все оценки были переведены в 0 и 1 (по порогу 1.5) и считается, что человек успешно справился с анкетированием, если он ответил верно (с точки зрения модели) на не менее чем 2 вопроса.

В случае предсказания непрерывной оценки предсказывается и используется только итоговая оценка. В таком случае можно считать, что, например, человек, итоговая оценка которого не ниже 6 баллов, успешно справился с вопросами анкетирования.

### Идеи по улучшению
На самом деле, я не так много успел сделать, но моя задача состояла в том, чтобы проверить, возможно, банальные, но простые и интуитивные решения, которые можно рассматривать как бейзлайн в дальнейшем при разработке более сложных моделей.

За время работы над заданием у меня появилось много разных идей, которые я мог бы опробовать будь у меня в распоряжении больше времени:
1) По сути самый большой минус в этой задаче — отсутствие большого количества данных: 650 ответов пользователей достаточно для построения простенькой модели классификации ответов с удовлетворительной точностью, но точно мало для создания серьезной модели с использованием, например, трансформерных эмбеддингов с хорошими показателями качества. По сути, датасет можно увеличить в разы, используя, например, техники промпт-инжиниринга для генерации данных с помощью предобученных языковых моделей через API (ChatGPT API, YandexGPT API) или используя открытые модели вроде Llama. При увеличении размеров датасета можно будет попробовать, например, модель классификации с использованием BERT и FFNN.
2) Также можно использовать техники промпт-инжиниринга, например, и для формирования эмбеддингов ответов — скорее всего, мощная языковая модель справится с этим сильно лучше любой модели, которую мы можем попробовать сами. А после можно сформировать векторную базу данных с разбивкой на кластеры и классифицировать новые эмбеддинги, например, с помощью косинусного расстояния. Что-то вроде этого: https://medium.com/@yu-joshua/efficient-similarity-search-clustering-of-dense-vectors-in-neo4j-b06c6f4bf9ce
3) Если говорить про более простые модели, которые еще можно опробовать, но я просто не успел физически, то можно, допустим, использовать:
	1) Многовыходная регрессия: чтобы не строить несколько моделей для каждого отдельного ответа, можно взять модель, которая сразу предсказывает несколько оценок (правда с учетом малого количества данных вряд ли она сможет хорошо обучиться).
	2) В разведывательном анализе было замечено, что есть связь между оценками одного и того человека — можно использовать это: стоить последовательно модели и в каждую следующую передавать информацию о предыдущих оценках, например, среднее значения предыдущих оценок (на самом деле, мельком опробовал этот подход на черновике — результатов особых не дал — но я думаю, что можно развить эту идею, хотя неизвестно, насколько хорошо “переобучаться“ на предыдущие оценки).