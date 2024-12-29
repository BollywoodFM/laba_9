# laba_9

Цель работы:
Научиться работать со средствами многопоточности языка C#.
Научиться работать с паттерном асинхронного программирования платформы .Net языка C#.
Задание №1
Создайте многопоточное приложение для получения средних цен акций за год. Используйте сайт https://www.marketdata.app/ для получения дневных котировок списка акций из файла ticker.txt. Формат ссылки следующий: https://api.marketdata.app/v1/stocks/candles/D/<Код_бумаги>/?from=<Начальная_дата >&to=<Конечная_дата>, где: Код_бумаги – тикер из списка акций Начальная_дата – метка времени начала запрашиваемого периода (год назад). Конечная_дата – метка времени конца запрашиваемого периода (текущая дата). Например, формат ссылки для AAPL: https://api.marketdata.app/v1/stocks/candles/D/AAPL/?from=2020-01-01&to=2020-01-03 Документация к API: https://www.marketdata.app/docs/api/stocks/candles Для запросов данных по всем тикерам из файла необходимо зарегистрироваться и сгенерировать токен для бесплатного доступа, который будет передаваться с каждым запросом. По мере получения данных выполните запуск задачи(Task), которая будет считать среднюю цену акции за год (используйте среднее значение для каждого дня как (High+Low)/2. Сложите все полученные значения и поделите на число дней). Результатом работы задачи будет являться среднее значение цены за год, которое необходимо вывести в файл в формате «Тикер:Цена». При этом обеспечьте потокобезопасный доступ к файлу между всеми задачами.
#Lb_09 (Program.cs)
```
using Newtonsoft.Json.Linq; // Библиотека для работы с JSON, используется для парсинга данных.
using System.ComponentModel; // Предоставляет классы для реализации компонентов и их свойств.
using System.Net.Http.Headers; // Для работы с заголовками HTTP-запросов.
using System.Text.Json; // Для сериализации и десериализации JSON.

public class StockData
{
    // Свойства, соответствующие полям JSON-ответа от API.
    public string s { get; set; } // Символ тикера (например, AAPL для Apple).
    public List<double> c { get; set; } // Список закрытых цен.
    public List<double> h { get; set; } // Список максимальных цен.
    public List<double> l { get; set; } // Список минимальных цен.
    public List<double> o { get; set; } // Список открытых цен.
    public List<int> t { get; set; } // Список меток времени.
    public List<int> v { get; set; } // Список объемов торгов.
}

class Market
{
    static readonly Mutex mutex = new Mutex(); // Мьютекс для синхронизации доступа к файлу при записи.

    // Асинхронное чтение данных из файла в список строк.
    static async Task ReadFileAsync(List<string> massive, string filePath)
    {
        using (StreamReader sr = new StreamReader(filePath))
        {
            string line;
            // Считываем файл построчно и добавляем каждую строку в список.
            while ((line = await sr.ReadLineAsync()) != null)
            {
                massive.Add(line);
            }
        }
    }

    // Запись строки в файл с использованием мьютекса.
    static void WriteToFile(string filePath, string text)
    {
        if (!File.Exists(filePath))
        {
            File.Create(filePath); // Создаем файл, если он не существует.
        }
        mutex.WaitOne(); // Захватываем мьютекс для предотвращения параллельной записи.
        try
        {
            File.AppendAllText(filePath, text + Environment.NewLine); // Добавляем текст в файл.
        }
        finally
        {
            mutex.ReleaseMutex(); // Освобождаем мьютекс.
        }
    }

    // Вычисление средней цены на основе максимальных и минимальных цен.
    static double calculateAvgPrice(List<double> highPrice, List<double> lowPrice)
    {
        double totalAvgPrice = 0;
        for (int i = 0; i < highPrice.Count; i++)
        {
            // Среднее арифметическое для каждой пары (максимальная и минимальная цена).
            totalAvgPrice += (highPrice[i] + lowPrice[i]) / 2;
        }
        return totalAvgPrice / highPrice.Count; // Возвращаем общее среднее значение.
    }

    // Получение данных с API и запись средней цены в файл.
    static async Task GetData(HttpClient client, string quote, string startDate, string endDate, string output)
    {
        // Формируем URL запроса.
        string apiKey = ""; // API-ключ для авторизации.
        string URL = $"https://api.marketdata.app/v1/stocks/candles/D/{quote}/?from={startDate}&to={endDate}&token={apiKey}";

        HttpClient cl = new HttpClient();
        HttpResponseMessage response = cl.GetAsync(URL).Result; // Выполняем запрос к API.

        if (!response.IsSuccessStatusCode)
        {
            // Если запрос завершился с ошибкой, выводим сообщение.
            Console.WriteLine($"Error! Status code: {response.StatusCode}");
        }
        else
        {
            // Парсим JSON-ответ от API.
            var json = await response.Content.ReadAsStringAsync();
            var data = JsonSerializer.Deserialize<StockData>(json);

            if (data != null && data.h != null && data.l != null)
            {
                // Если данные успешно получены, вычисляем среднюю цену.
                double avgPrice = calculateAvgPrice(data.h, data.l);
                string result = $"{quote}:{avgPrice}"; // Формируем строку для записи в файл.
                WriteToFile(output, result); // Записываем результат в файл.
                Console.WriteLine($"Average price for {quote}: {avgPrice}"); // Выводим результат.
            }
            else
            {
                // Если данных недостаточно, выводим сообщение.
                Console.WriteLine($"Not enough data for {quote}");
            }
        }
    }

    // Основной метод программы.
    static async Task Main(string[] args)
    {
        string tickerPath = "C:\\Users\\Фукс\\Desktop\\C#\\Lb_3_9\\Lb_09\\files\\ticker.txt"; // Путь к файлу с тикерами.
        string outputPath = "C:\\Users\\Фукс\\Desktop\\C#\\Lb_3_9\\Lb_09\\files\\output.txt"; // Путь к файлу для записи результата.

        List<string> ticker = []; // Список для хранения тикеров.
        await ReadFileAsync(ticker, tickerPath); // Считываем тикеры из файла.

        // Формируем даты для запросов (прошлый год).
        DateTime endDateTime = DateTime.Now;
        string endDate = endDateTime.AddMonths(-1).ToString("yyyy-MM-dd"); // Конечная дата — месяц назад.
        string startDate = endDateTime.AddYears(-1).AddMonths(1).ToString("yyyy-MM-dd"); // Начальная дата — год назад.

        Console.WriteLine($"Start Date: {startDate}"); // Вывод начальной даты.
        Console.WriteLine($"End Date: {endDate}"); // Вывод конечной даты.

        HttpClient client = new HttpClient(); // Инициализация клиента HTTP.
        client.DefaultRequestHeaders.Clear(); // Очистка заголовков запроса.
        client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json")); // Устанавливаем тип контента.

        List<Task> tasks = new List<Task>(); // Список задач для параллельных запросов.

        // Для каждого тикера формируем запрос к API.
        foreach (var quote in ticker)
        {
            tasks.Add(GetData(client, quote, startDate, endDate, outputPath)); // Добавляем задачу в список.
        }
        await Task.WhenAll(tasks); // Ожидаем завершения всех задач.
    }
}
```
Задание №2
На основе Лабораторной работы №6 создайте графическое приложение, получающее текущую погоду в разных городах мира. Используйте WinForms или WPF. Загрузите список городов с координатами из файла city.txt. Добавьте в интерфейс приложения 2 элемента – один для отображения списка городов, второй элемент – кнопку, по нажатию на которую происходит загрузка текущей погоды в выбранном по API из Лабораторной работы №6. Используйте ключевые слова async/await для загрузки данных и обновления интерфейса приложения таким образом, чтобы графический интерфейс не блокировался на время загрузки. Если разработка ведется на *NIX или MACOS, допускается использовать консольное приложение, тем не менее следует использовать ключевые слова async/await при реализации методов.
WeatherApp (Form1.cs)
```
using System;
// Подключаем библиотеки, необходимые для работы с коллекциями, файловой системой, HTTP-запросами, обработкой JSON и формами Windows Forms.
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.IO;
using System.Linq;
using System.Net.Http.Headers;
using System.Net.Http;
using System.Text.Json;
using System.Text.Json.Nodes;
using System.Threading.Tasks;
using System.Windows.Forms;
using System.Security.Policy;

namespace WeatherApp
{
    // Главная форма приложения, которая управляет пользовательским интерфейсом
    public partial class Form1 : Form
    {
        // API ключ для доступа к OpenWeatherMap
        private readonly string apiKey = "";

        // URL для получения данных о погоде
        private readonly string apiUrl = "https://api.openweathermap.org/data/2.5/weather";

        // Путь к файлу, содержащему данные о городах
        private readonly string cityPath = "C:\\Users\\Фукс\\Desktop\\C#\\Lb_3_9\\Lb_09\\files\\city.txt";

        // Список объектов городов
        private readonly List<City> _cities = new List<City>();

        // Конструктор формы, вызывается при ее создании
        public Form1()
        {
            InitializeComponent(); // Инициализирует компоненты формы
            LoadCitiesAsync(cityPath); // Асинхронно загружает города из файла
        }

        // Метод для асинхронной загрузки данных о городах
        private async Task LoadCitiesAsync(string filePath)
        {
            try
            {
                // Читает все строки из файла
                var cities = File.ReadAllLines(filePath);

                foreach (var city in cities)
                {
                    // Разделяет строку на название города и координаты
                    var parts = city.Split('\t');
                    if (parts.Length == 2)
                    {
                        var name = parts[0];
                        var coord = parts[1].Replace(" ", "").Split(',');

                        // Преобразует координаты в числовой формат
                        var latitude = Convert.ToDouble(coord[0].Replace(".", ","));
                        var longitude = Convert.ToDouble(coord[1].Replace(".", ","));

                        // Создает объект City и добавляет в список
                        var info = new City(name, latitude, longitude);
                        _cities.Add(info);
                    }
                }

                // Заполняет выпадающий список ComboBox данными из списка городов
                CityComboBox.DataSource = _cities;
            }
            catch (Exception ex)
            {
                // Отображает сообщение об ошибке, если что-то пошло не так
                MessageBox.Show($"Error loading cities: {ex.Message}");
            }
        }

        // Обработчик события нажатия на кнопку "Get Weather"
        private async void GetWeatherButton_Click(object sender, EventArgs e)
        {
            // Проверяет, выбран ли город в выпадающем списке
            if (CityComboBox.SelectedItem is City selectedCity)
            {
                try
                {
                    // Получает данные о погоде для выбранного города
                    var weather = await FetchWeatherAsync(apiUrl, selectedCity);

                    if (weather != null)
                    {
                        // Если данные успешно получены, отображает их в текстовом поле
                        ResulttextBox.Text = weather.ToString();
                    }
                    else
                    {
                        // Если данные не удалось получить, выводит сообщение
                        MessageBox.Show("Failed fetching weather data. Try again later.");
                    }
                }
                catch (Exception ex)
                {
                    // Обрабатывает исключения и выводит сообщение об ошибке
                    MessageBox.Show($"Error occurred: {ex.Message}");
                }
            }
            else
            {
                // Если город не выбран, просит пользователя выбрать его
                MessageBox.Show("Please choose a city");
            }
        }

        // Метод для получения данных о погоде с API
        private async Task<Weather> FetchWeatherAsync(string URL, City city)
        {
            try
            {
                // Создает HTTP-клиент для отправки запросов
                HttpClient client = new HttpClient
                {
                    BaseAddress = new Uri(URL)
                };

                // Очищает заголовки клиента и задает, что ожидаем ответ в формате JSON
                client.DefaultRequestHeaders.Clear();
                client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

                // Формирует URL с параметрами (координаты города, API-ключ и единицы измерения)
                var urlParameters = $"?lat={city.Latitude}&lon={city.Longitude}&appid={apiKey}&units=metric";
                var fullUrl = URL + urlParameters;

                // Отправляет GET-запрос к API
                var response = await client.GetAsync(fullUrl);

                if (response.IsSuccessStatusCode)
                {
                    // Если запрос успешен, читает содержимое ответа как строку
                    var responseString = await response.Content.ReadAsStringAsync();

                    // Парсит строку JSON в объект
                    var json = JsonObject.Parse(responseString);

                    // Извлекает необходимые данные и создает объект Weather
                    Weather res = new Weather
                    {
                        Country = (string)json["sys"]["country"],
                        Name = (string)json["name"],
                        Temp = (double)json["main"]["temp"],
                        Description = (string)json["weather"][0]["main"]
                    };

                    return res;
                }
                else
                {
                    // Если запрос завершился с ошибкой, выводит сообщение
                    MessageBox.Show($"API error: {response.StatusCode}");
                }
                return null;
            }
            catch (Exception ex)
            {
                // Обрабатывает исключения и выводит сообщение об ошибке
                MessageBox.Show($"Failed fetching weather data: {ex.Message}");
                return null;
            }
        }
    }

    // Класс для представления города
    public class City
    {
        public string Name { get; }
        public double Latitude { get; }
        public double Longitude { get; }

        public City(string name, double latitude, double longitude)
        {
            Name = name;
            Latitude = latitude;
            Longitude = longitude;
        }

        public override string ToString()
        {
            return Name; // Отображает название города в выпадающем списке
        }
    }

    // Класс для представления данных о погоде
    public class Weather
    {
        public string Country { get; set; }
        public string Name { get; set; }
        public double Temp { get; set; }
        public string Description { get; set; }

        // Конструктор с параметрами
        public Weather(string country, string name, double temp, string description)
        {
            Country = country;
            Name = name;
            Temp = temp;
            Description = description;
        }

        // Пустой конструктор для удобства
        public Weather()
        {
        }

        public override string ToString()
        {
            // Форматированный вывод данных о погоде
            return $"Country: {Country}, City: {Name}, Temperature: {Temp} °C, Description: {Description}";
        }
    }
}
```
#WeatherApp (Form1.Designer.cs)
```
namespace WeatherApp
{
    partial class Form1
    {
        /// <summary>
        /// Обязательная переменная конструктора.
        /// </summary>
        private System.ComponentModel.IContainer components = null;

        /// <summary>
        /// Освободить все используемые ресурсы.
        /// </summary>
        /// <param name="disposing">истинно, если управляемый ресурс должен быть удален; иначе ложно.</param>
        protected override void Dispose(bool disposing)
        {
            if (disposing && (components != null))
            {
                components.Dispose();
            }
            base.Dispose(disposing);
        }

        #region Код, автоматически созданный конструктором форм Windows

        /// <summary>
        /// Требуемый метод для поддержки конструктора — не изменяйте 
        /// содержимое этого метода с помощью редактора кода.
        /// </summary>
        private void InitializeComponent()
        {
            this.CityComboBox = new System.Windows.Forms.ComboBox();
            this.GetWeatherButton = new System.Windows.Forms.Button();
            this.ResulttextBox = new System.Windows.Forms.TextBox();
            this.SuspendLayout();
            // 
            // CityComboBox
            // 
            this.CityComboBox.FormattingEnabled = true;
            this.CityComboBox.Location = new System.Drawing.Point(190, 224);
            this.CityComboBox.Name = "CityComboBox";
            this.CityComboBox.Size = new System.Drawing.Size(130, 21);
            this.CityComboBox.TabIndex = 0;
            // 
            // GetWeatherButton
            // 
            this.GetWeatherButton.Location = new System.Drawing.Point(630, 224);
            this.GetWeatherButton.Name = "GetWeatherButton";
            this.GetWeatherButton.Size = new System.Drawing.Size(95, 21);
            this.GetWeatherButton.TabIndex = 1;
            this.GetWeatherButton.Text = "Get Weather";
            this.GetWeatherButton.UseVisualStyleBackColor = true;
            this.GetWeatherButton.Click += new System.EventHandler(this.GetWeatherButton_Click);
            // 
            // ResulttextBox
            // 
            this.ResulttextBox.Location = new System.Drawing.Point(328, 318);
            this.ResulttextBox.Multiline = true;
            this.ResulttextBox.Name = "ResulttextBox";
            this.ResulttextBox.Size = new System.Drawing.Size(284, 111);
            this.ResulttextBox.TabIndex = 2;
            // 
            // Form1
            // 
            this.AutoScaleDimensions = new System.Drawing.SizeF(6F, 13F);
            this.AutoScaleMode = System.Windows.Forms.AutoScaleMode.Font;
            this.ClientSize = new System.Drawing.Size(1211, 482);
            this.Controls.Add(this.ResulttextBox);
            this.Controls.Add(this.GetWeatherButton);
            this.Controls.Add(this.CityComboBox);
            this.Name = "Form1";
            this.Text = "Form1";
            this.ResumeLayout(false);
            this.PerformLayout();

        }

        #endregion

        private System.Windows.Forms.ComboBox CityComboBox;
        private System.Windows.Forms.Button GetWeatherButton;
        private System.Windows.Forms.TextBox ResulttextBox;
    }
}
```
WeatherApp (Program.cs)
```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace WeatherApp
{
    internal static class Program
    {
        /// <summary>
        /// 
        /// </summary>
        [STAThread]
        static void Main()
        {
            Application.EnableVisualStyles();
            Application.SetCompatibleTextRenderingDefault(false);
            Application.Run(new Form1());
        }
    }
}
```
ConsoleApp2 (Program.cs)
```
// Указываем путь к файлу, содержащему данные о городах
string cityPath = "C:\\Users\\Фукс\\Desktop\\C#\\Lb_3_9\\Lb_09\\files\\city.txt";

// Асинхронный метод для загрузки данных о городах из файла
async Task LoadCitiesAsync(string filePath)
{
    // Попытка чтения данных из файла
    try
    {
        // Считываем все строки из файла в массив строк
        var cities = File.ReadAllLines(filePath);

        // Проходим по каждой строке файла
        foreach (var city in cities)
        {
            // Разделяем строку на части, используя табуляцию (\t) в качестве разделителя
            var parts = city.Split('\t');

            // Берем первую часть строки — название города
            var name = parts[0];

            // Берем вторую часть строки — координаты (широта и долгота),
            // и разделяем их по запятой с пробелом
            var coord = parts[1].Split(", ");

            // Для отладки выводим долготу на экран
            Console.WriteLine($"'{coord[1]}'");

            // Конвертируем широту из строки в число, заменяя точку на запятую
            // (это нужно для правильной обработки чисел в некоторых локалях)
            Console.WriteLine($"Hey ----{Convert.ToDouble(coord[0].Replace('.', ','))}");

            // Аналогично для долготы
            Console.WriteLine($"Hey ----{Convert.ToDouble(coord[1].Replace('.', ','))}");

            // Выводим всю строку целиком для отладки
            Console.WriteLine(city);

            // Выводим имя города и координаты
            Console.WriteLine(name, coord);

            // Преобразуем широту и долготу в числа с плавающей запятой
            var latitude = Convert.ToDouble(coord[0].Trim());
            var longitude = Convert.ToDouble(coord[1].Trim());

            // Выводим название города и его координаты
            Console.WriteLine(name, latitude, longitude);
        }
    }
    // Обрабатываем исключения, если они возникнут при чтении файла или обработке данных
    catch (Exception ex)
    {
        Console.WriteLine($"Error loading cities: {ex.Message}");
    }
}

// Вызываем метод для загрузки данных о городах, передавая путь к файлу
LoadCitiesAsync(cityPath);
```
