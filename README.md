Client:

using Common;
using System;
using System.Collections.Generic;
using System.Linq;
using System.ServiceModel;
using System.Text;
using System.Threading.Tasks;

namespace Client
{
    public class Program
    {
        static void Main(string[] args)
        {
            ChannelFactory<ILibrary> factory = new ChannelFactory<ILibrary>("BookReview");
            ILibrary proxy = factory.CreateChannel();

            proxy.Subscribe();
            PrintAllBooks(proxy);

            Console.WriteLine("Input book title as title.txt");
            string fileName = Console.ReadLine();
            try
            {
                proxy.ChangeScore(fileName, 5);
            }
            catch (FaultException<CustomException> ex)
            {
                Console.WriteLine($"ERROR : {ex.Detail.Message}");
            }

            PrintAllBooks(proxy);
            proxy.Unsubscribe();

            Console.ReadKey();
        }

        public static void PrintAllBooks(ILibrary proxy)
        {
            Console.WriteLine("<Press any key for book overview>");
            Console.ReadKey();

            Console.WriteLine("Book Overview:");
            foreach (Book book in proxy.GetAllBooks())
            {
                Console.WriteLine($"Title: {book.Title}  score: {book.Score}");
            }
            Console.WriteLine();
        }
    }
}


Common:
using System;
using System.Collections.Generic;
using System.Linq;
using System.Runtime.Serialization;
using System.Text;
using System.Threading.Tasks;

namespace Common
{
    public delegate void BookScoreHandler(object sender, BookEventArgs e);

    [DataContract]
    public class Book
    { 
        private string title;
        private int score;

        public Book(string title)
        {
            this.title = title;
            this.score = 0;
        }

        public Book() { }

        [DataMember]
        public string Title { get { return title; } set { title = value; } }

        [DataMember]
        public int Score { get { return score; } set { score = value; } }
    }
}

using System;
using System.Collections.Generic;
using System.Linq;
using System.Runtime.Serialization;
using System.Text;
using System.Threading.Tasks;

namespace Common
{
    public class BookEventArgs:EventArgs
    {
        private string title;
        private int oldScore;
        private int newScore;

        public string Title { get { return title; } set { title = value; } }
        public int OldScore { get { return oldScore; } set { oldScore = value; } }
        public int NewScore { get { return newScore; } set { newScore = value; } }  

        public BookEventArgs(string title, int oldScore, int newScore)
        {
            this.title = title;
            this.oldScore = oldScore;
            this.newScore = newScore;
        }
    }
}

using System;
using System.Collections.Generic;
using System.Linq;
using System.Runtime.Serialization;
using System.Text;
using System.Threading.Tasks;

namespace Common
{
    [DataContract]
    public class CustomException
    {
        string message;

        public CustomException(string message)
        {
            this.Message = message;
        }

        [DataMember]
        public string Message { get => message; set => message = value; }
    }
}

using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Runtime.Serialization;
using System.Text;
using System.Threading.Tasks;

namespace Common
{
    [DataContract]
    public class FileManipulationOptions : IDisposable
    {
        public FileManipulationOptions(MemoryStream memomoryStream, string fileName)
        {
            this.MemomoryStream = memomoryStream;
            this.FileName = fileName;
        }

        [DataMember]
        public MemoryStream MemomoryStream { get; set; }

        [DataMember]
        public string FileName { get; set; }

        public void Dispose()
        {
            if (MemomoryStream == null)
                return;
            try
            {
                MemomoryStream.Dispose();
                MemomoryStream.Close();
                MemomoryStream = null;
            }
            catch (Exception)
            {
                Console.WriteLine("Unsuccesful disposing!");
            }
        }
    }
}


using System;
using System.Collections.Generic;
using System.Linq;
using System.ServiceModel;
using System.Text;
using System.Threading.Tasks;

namespace Common
{
    [ServiceContract]
    public interface ILibrary
    {
        [OperationContract]
        [FaultContract(typeof(CustomException))]
        void AddBookRecommendation(FileManipulationOptions options);

        [OperationContract]
        List<Book> GetAllBooks();

        [OperationContract]
        [FaultContract(typeof(CustomException))]
        void ChangeScore(string title, int score);

        [OperationContract]
        void Subscribe();

        [OperationContract]
        void Unsubscribe();
    }
}


using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Common
{
    public class LibrarySubscriber
    {
        public void OnRaitingChanged(object sender, BookEventArgs e)
        {
            Console.WriteLine($"Book {e.Title} score changed from {e.OldScore} to {e.NewScore} ");
        }
    }
}


Server:
using Common;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Server
{
    public class Database
    {
        readonly static Dictionary<string, Book> collectionOfBooks;

        static Database()
        {
            collectionOfBooks = new Dictionary<string, Book>();
        }

        public static Dictionary<string, Book> CollectionOfBooks { get { return collectionOfBooks; } }
    }
}


using Common;
using System;
using System.Collections.Generic;
using System.Configuration;
using System.IO;
using System.Linq;
using System.ServiceModel;
using System.Text;
using System.Threading.Tasks;

namespace Server
{
    public class LibraryService : ILibrary
    {
        private readonly LibrarySubscriber subscriber = new LibrarySubscriber();
        public event BookScoreHandler BookScoreChanged;

        private readonly string fileDirectoryPath = ConfigurationManager.AppSettings["bookPath"];

        #region AddBookRecommendation
        [OperationBehavior(AutoDisposeParameters = true)]
        public void AddBookRecommendation(FileManipulationOptions options)
        {
            try
            {
                if (!Directory.Exists(fileDirectoryPath))
                {
                    Directory.CreateDirectory(fileDirectoryPath);
                }

                if (options.MemomoryStream == null || options.MemomoryStream.Length == 0)
                {
                    throw new FaultException<CustomException>(new CustomException($"No content provided for book {options.FileName}"));
                }

                SaveFile(options.MemomoryStream, $"{fileDirectoryPath}/{options.FileName}");
            }
            catch (Exception ex)
            {
                throw new FaultException<CustomException>(new CustomException(ex.Message));
            }
        }

        private void SaveFile(MemoryStream memoryStream, string filePath)
        {
            using (FileStream fileStream = new FileStream($"{filePath}", FileMode.Create, FileAccess.Write))
            {
                memoryStream.WriteTo(fileStream);
                fileStream.Dispose();
                memoryStream.Dispose();
            }

        }
        #endregion method

        #region GetAllBooks
        private void DatabaseSetUp()
        {
                string[] files = Directory.GetFiles(fileDirectoryPath);
                foreach (string filePath in files)
                {
                    Database.CollectionOfBooks.Add(Path.GetFileName(filePath), new Book(Path.GetFileName(filePath).Split('.')[0]));
                }
        }

        public List<Book> GetAllBooks()
        {
            if(Database.CollectionOfBooks.Count == 0)
            {
                DatabaseSetUp();
            }
           
            return new List<Book>(Database.CollectionOfBooks.Values);
        }
        #endregion


        public void ChangeScore(string title, int score)
        {
            if(Database.CollectionOfBooks.TryGetValue(title, out Book book))
            {
                if(BookScoreChanged != null)
                {
                    BookScoreChanged(this, new BookEventArgs(book.Title, book.Score, score));
                    book.Score = score;
                }
                return;
            }
            throw new FaultException<CustomException>(new CustomException($"Book {title} not found"));

        }

        public void Subscribe()
        {
            BookScoreChanged += subscriber.OnRaitingChanged;
        }

        public void Unsubscribe()
        {
            BookScoreChanged -= subscriber.OnRaitingChanged;
        }
    }
}



using Common;
using System;
using System.Collections.Generic;
using System.Linq;
using System.ServiceModel;
using System.Text;
using System.Threading.Tasks;

namespace Server
{
    public class Program
    {
        static void Main(string[] args)
        {
            using (ServiceHost host = new ServiceHost(typeof(LibraryService)))
            {
                host.Open();

                Console.WriteLine("Service is open, press any key to close it");
                Console.ReadKey();

                host.Close();
            }

            Console.WriteLine("Service is closed");
            Console.ReadKey();
        }
    }
}


Upload Client:

using Common;
using System;
using System.Collections.Generic;
using System.Configuration;
using System.IO;
using System.Linq;
using System.ServiceModel;
using System.Text;
using System.Threading.Tasks;

namespace UploadClient
{
    public class BookRecommendation
    {
        private FileSystemWatcher fileWatcher;
        private readonly ILibrary proxy;
        
        public BookRecommendation(string path, ILibrary sender)
        {
            CreateFileSystemWatcher(path);
            this.proxy = sender;
        }

        public void CreateFile(string uploadPath)
        {
            Console.WriteLine("Enter book title as title.txt: ");
            string bookTitle = Console.ReadLine();
            try
            {
                using (var fs = File.Create(Path.Combine(uploadPath, bookTitle)))
                {
                }
            }
            catch (Exception e)
            {
                Console.WriteLine($"ERROR : {e}");
            }
        }

        private void CreateFileSystemWatcher(string path)
        {
            fileWatcher = new FileSystemWatcher()
            {
                Path = path,
                Filter = "*.*",
                NotifyFilter = NotifyFilters.LastWrite
            };

            fileWatcher.Changed += FileChanged;
            fileWatcher.EnableRaisingEvents = true;
        }

        private void FileChanged(object sender, FileSystemEventArgs e)
        {
            try
            {
                SendFile(e.FullPath, e.Name);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"ERROR : {ex}");
            }
        }

        private void SendFile(string filePath, string fileName)
        {
            try
            {
                proxy.AddBookRecommendation(new FileManipulationOptions(GetMemoryStream(filePath), fileName));
                Console.WriteLine($"Recommondation for {filePath} modified at {DateTime.Now}");
            }
            catch (FaultException<CustomException> ex)
            {
                Console.WriteLine($"ERROR : {ex.Detail.Message}");
            }

        }

        public static MemoryStream GetMemoryStream(string filePath)
        {
            
            if (!File.Exists(filePath))
            {
                Console.WriteLine($"Cannot process the file because directory not exists.");
                return null;
            }

            MemoryStream memoryStream = new MemoryStream();
            FileStream fileStream = null;
            try
            {
                fileStream = new FileStream(filePath, FileMode.Open, FileAccess.Read);
                fileStream.CopyTo(memoryStream);
                fileStream.Close();

            }
            catch (IOException e)
            {
                Console.WriteLine($"Cannot process the file {filePath}. Message: {e.Message}");
            }
            catch (Exception e)
            {
                Console.WriteLine($"Something went wrong: {filePath}.  Message: {e.Message}");
            }
            finally
            {
                if (fileStream != null)
                {
                    fileStream.Dispose();
                }
            }
            return memoryStream;
        }
    }
}


using Common;
using System;
using System.Collections.Generic;
using System.Configuration;
using System.IO;
using System.Linq;
using System.ServiceModel;
using System.Text;
using System.Threading.Tasks;

namespace UploadClient
{
    public class Program
    {
        static void Main(string[] args)
        {
            ChannelFactory<ILibrary> factory = new ChannelFactory<ILibrary>("Library");
            ILibrary proxy = factory.CreateChannel();

            var uploadPath = ConfigurationManager.AppSettings["uploadPath"];

            if (!Directory.Exists(uploadPath))
            {
                Directory.CreateDirectory(uploadPath);
            }

            BookRecommendation bookRecommendation = new BookRecommendation(uploadPath, proxy);

            int selectedNumber;
            do
            {
                selectedNumber = PrintMenu();
                switch (selectedNumber)
                {
                    case 0:
                        Console.WriteLine("You need to select existing option");
                        break;
                    case 1:
                        bookRecommendation.CreateFile(uploadPath);
                        break;
                }
            }
            while (selectedNumber != 2);
            Console.ReadKey();
        }

        static int PrintMenu()
        {
            Console.WriteLine("Select an option");
            Console.WriteLine("1. Add book recommendation");
            Console.WriteLine("2. Exit and provide content for recommendations");
            if (Int32.TryParse(Console.ReadLine(), out int number))
            {
                if (number >= 1 && number <= 2)
                {
                    return number;
                }
            }
            return 0;
        }
    }
}










