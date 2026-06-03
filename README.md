#include <iostream>
#include <string>
#include <vector>    //在内存中存储数据
#include <map>
#include <fstream>   //读写硬盘上的文件
#include <sstream>   //在字符串和各种数据之间进行转换
#include <iomanip>   //输入输出操作符
#include <ctime>     //提供时间与日期相关的函数
#include <algorithm> //提供一系列通用的算法，如查找，排序，删除等。

using namespace std;

// ===================== 工具类 (Utils) =====================
class Utils {
public:         //获取时间，方便用于计算借阅和逾期天数及罚款 
    // 获取当前日期字符串 (YYYY-MM-DD)
    static string getCurrentDate() {
        time_t now = time(0);
        tm* ltm = localtime(&now);
        char buffer[80];
        strftime(buffer, 80, "%Y-%m-%d", ltm); //格式化时间输出
        return string(buffer);
    }

    // 计算两个日期的天数差 (简单实现，用于逾期判断)
    static int dateDiff(const string& date1, const string& date2) {
        int y1, m1, d1, y2, m2, d2;
        (void)sscanf(date1.c_str(), "%d-%d-%d", &y1, &m1, &d1);
        (void)sscanf(date2.c_str(), "%d-%d-%d", &y2, &m2, &d2);//将格式化时间输出转为独立的时间输出
        // 简化算法：仅算年份差*365 + 月份差*30 + 天差
        return (y2 - y1) * 365 + (m2 - m1) * 30 + (d2 - d1);
    }

    // 暂停控制台并提示按回车继续
    static void pauseConsole() {
        cout << "\n请按回车键继续...";
        cin.ignore(numeric_limits<streamsize>::max(), '\n');//清空当前输入缓冲区的一整行内容
        cin.get();//等待用户输入
    }

    // 清除输入缓冲区，防止非法输入导致死循环
    static void clearInputBuffer() {
        cin.clear();
        cin.ignore(numeric_limits<streamsize>::max(), '\n');
    }
};

// ===================== 实体类 (Entities) =====================

// 图书类
class Book {
private:
    string id;
    string title;
    string author;
    int stock;          // 总库存
    int available;      // 可借数量
public:
    Book() : stock(0), available(0) {}
    Book(string i, string t, string a, int s) : id(i), title(t), author(a), stock(s), available(s) {}

    string getId() const { return id; }
    string getTitle() const { return title; }
    string getAuthor() const { return author; }
    int getStock() const { return stock; }
    int getAvailable() const { return available; }
    bool isAvailable() const { return available > 0; }

    void setInfo(string t, string a, int s) {
        title = t; author = a; stock = s; available = s - (stock - available);
    }

    // 借出与归还逻辑
    void borrow() { if (available > 0) available--; }
    void returnBook() { if (available < stock) available++; }

    // 转换为字符串用于文件存储
    string toString() const {
        return id + "," + title + "," + author + "," + to_string(stock) + "," + to_string(available);//字符串拼接，在每一个属性前都加了一个，
    }

    // 从字符串解析，数据反序列化
    void fromString(const string& data) {
        stringstream ss(data);
        
        getline(ss, id, ',');
        getline(ss, title, ','); 
        getline(ss, author, ',');   //从字符串流中读取字符，一直读到，为止，把读取到底内容存进id里并自动丢弃，
        
        ss >> stock;  //在读取完，之后，光标正好停在库存数字前面（在stock和available之间的，上）
        ss.ignore(1);   //跳过这个，
        ss >> available; //将剩余的可借数量读入available中
    }
};

// 用户基类
class User {
protected:
    string id;
    string name;
    string password;
    string role; // "admin" or "reader"
public:
    User() {}
    User(string i, string n, string p, string r) : id(i), name(n), password(p), role(r) {}

    string getId() const { return id; }
    string getName() const { return name; }
    string getPassword() const { return password; }
    string getRole() const { return role; }

    virtual void showMenu(class LibrarySystem& sys) = 0; // 纯虚函数，多态体现，在两个子类中均有重写

    string toString() const { return id + "," + name + "," + password + "," + role; }//与上文同理，字符串拼接
    void fromString(const string& data) {
        stringstream ss(data);
        getline(ss, id, ','); 
        getline(ss, name, ','); 
        getline(ss, password, ','); 
        getline(ss, role, ',');
    }
};

// 读者类
class Reader : public User {
public:
    Reader() { role = "reader"; }
    Reader(string i, string n, string p) : User(i, n, p, "reader") {}

    void showMenu(class LibrarySystem& sys) override;
};

// 管理员类
class Admin : public User {
public:
    Admin() { role = "admin"; }
    Admin(string i, string n, string p) : User(i, n, p, "admin") {}

    void showMenu(class LibrarySystem& sys) override;
};

// 借阅记录类
class BorrowRecord {
private:
    string recordId;
    string bookId;
    string readerId;
    string borrowDate;
    string returnDate;   // 空表示未归还
    bool isReturned;
public:
    BorrowRecord() : isReturned(false) {}//创建一个BorrowRecord对象，如果他为空，就自动把他的isReturned设为false。
    BorrowRecord(string rid, string bid, string uid, string bDate)
        : recordId(rid), bookId(bid), readerId(uid), borrowDate(bDate), returnDate(""), isReturned(false) {
    }

    string getBookId() const { return bookId; }
    string getReaderId() const { return readerId; }
    string getBorrowDate() const { return borrowDate; }
    bool getIsReturned() const { return isReturned; }

    void setReturned(string rDate) { returnDate = rDate; isReturned = true; }//填上归还日期，并把状态改成已归还。

    string toString() const {
        return recordId + "," + bookId + "," + readerId + "," + borrowDate + "," + returnDate + "," + (isReturned ? "1" : "0");
    }//序列化，用“，“把各个数据拼接起来。
    void fromString(const string& data) {
        stringstream ss(data);
        getline(ss, recordId, ','); getline(ss, bookId, ','); getline(ss, readerId, ',');
        getline(ss, borrowDate, ','); getline(ss, returnDate, ',');
        int status; ss >> status; isReturned = (status == 1);
    }//反序列化，与上文配合形成数据持久化。
};

// ===================== 核心系统类 (LibrarySystem) =====================
class LibrarySystem {
private:
    vector<Book> books;//动态数组
    map<string, User*> users; // key: userId，存储用户信息
    vector<BorrowRecord> records;//借阅记录的动态数组
    User* currentUser;//指向user对象的指针

    const string BOOK_FILE = "books.txt";
    const string USER_FILE = "users.txt";
    const string RECORD_FILE = "records.txt";

public:
    LibrarySystem() : currentUser(nullptr) {
        loadData();
        // 如果没有管理员，初始化一个默认管理员
        if (users.find("admin") == users.end()) {
            users["admin"] = new Admin("admin", "系统管理员", "123456");
        }
        // 检查是否存在某个特定ID的读者，若不存在则创建
        if (users.find("reader") == users.end()) {
            users["reader"] = new Reader("reader", "张三", "reader123");
        }
    }

    ~LibrarySystem() {
        saveData();
        for (auto& pair : users) delete pair.second;
    }

    // 数据持久化加载
    void loadData() {
        loadBooks(); loadUsers(); loadRecords();
    }

    void loadBooks() {
        ifstream file(BOOK_FILE);//打开文件
        if (!file.is_open()) return;
        string line;
        while (getline(file, line)) {
            Book b; b.fromString(line);
            books.push_back(b);
        }//循环读取，每次读取一整行的文字
        file.close();//读取完成，关闭文件
    }

    void loadUsers() {
        ifstream file(USER_FILE);
        if (!file.is_open()) return;
        string line;
        while (getline(file, line)) {
            User* u = nullptr;
            stringstream ss(line);
            string id, name, pass, role;
            getline(ss, id, ','); getline(ss, name, ','); getline(ss, pass, ','); getline(ss, role, ',');
            if (role == "admin") u = new Admin(id, name, pass);
            else u = new Reader(id, name, pass);
            users[id] = u;
        }
        file.close();
    }

    void loadRecords() {
        ifstream file(RECORD_FILE);
        if (!file.is_open()) return;
        string line;
        while (getline(file, line)) {
            BorrowRecord r; r.fromString(line);
            records.push_back(r);
        }
        file.close();
    }

    // 数据持久化保存
    void saveData() {
        saveBooks(); saveUsers(); saveRecords();
    }

    void saveBooks() {
        ofstream file(BOOK_FILE);//覆盖文件
        for (const auto& b : books) file << b.toString() << endl;//循环写入新的数据
        file.close();//关闭文件
    }

    void saveUsers() {
        ofstream file(USER_FILE);
        for (const auto& pair : users) file << pair.second->toString() << endl;
        file.close();
    }

    void saveRecords() {
        ofstream file(RECORD_FILE);
        for (const auto& r : records) file << r.toString() << endl;
        file.close();
    }

    // 登录验证
    bool login() {
        string id, pass;
        cout << "\n========== 用户登录 ==========\n";
        cout << "请输入用户ID: "; cin >> id;
        cout << "请输入密码: "; cin >> pass;

        auto it = users.find(id);
        if (it != users.end() && it->second->getPassword() == pass) {
            currentUser = it->second;
            cout << "登录成功！欢迎您，" << currentUser->getName() << endl;
            return true;
        }//存在且密码正确才能进入
        cout << "登录失败：用户名或密码错误。\n";
        return false;
    }

    // 查找图书辅助函数
    Book* findBook(string bookId) {
        for (auto& b : books) {
            if (b.getId() == bookId) return &b;
        }
        return nullptr;
    }

    // 业务逻辑：借书
    bool borrowBook(string bookId, string readerId) {
        Book* book = findBook(bookId);
        if (!book || !book->isAvailable()) return false;//没有这本书或者这本书被借完

        // 检查该读者是否已达借阅上限 (最多借5本)
        int borrowedCount = 0;
        for (const auto& r : records) {
            if (r.getReaderId() == readerId && !r.getIsReturned()) borrowedCount++;
        }
        if (borrowedCount >= 5) return false;

        book->borrow();
        string rid = "R" + to_string(time(0)); // 简单的记录ID生成:把数字转换成字符串，并在前边加个R
        records.emplace_back(rid, bookId, readerId, Utils::getCurrentDate());//建立一个新的BorrowRecord对象，并放在records的最后边
        return true;
    }

    // 业务逻辑：还书
    bool returnBook(string bookId) {
        Book* book = findBook(bookId);
        if (!book) return false;

        // 查找该图书最近的一条未归还记录
        for (auto& r : records) {
            if (r.getBookId() == bookId && !r.getIsReturned()) {
                r.setReturned(Utils::getCurrentDate());//将当前的日期作为归还日期写入
                book->returnBook();
                return true;
            }
        }
        return false;
    }

    // 菜单入口
    void run() {
        while (true) {
            if (!currentUser) {         //判断当前有没有人登录
                if (!login()) {
                    cout << "1. 重新登录\n2. 退出系统\n请选择: ";
                    int c; cin >> c;
                    if (c == 2) break;
                }
            }
            else {
                currentUser->showMenu(*this);//登陆成功后，把控制权交给用户
                currentUser = nullptr; // 退出登录后重置
            }
        }
    }

    // ================== 管理员功能实现 ==================
    void adminManageBooks() {
        int choice; 
        string id, title, author; 
        int stock;
        while (true) {
            cout << "\n=== 图书管理 ===\n1. 添加图书\n2. 删除图书\n3. 修改图书\n4. 查询图书\n0. 返回\n选择: ";
            cin >> choice;
            if (choice == 0) break;//选0则跳出循环

            switch (choice) {
            case 1:
                cout << "输入ISBN/ID: "; cin >> id;
                if (findBook(id)) { cout << "图书已存在！\n"; break; }
                cout << "输入书名: "; cin >> title;
                cout << "输入作者: "; cin >> author;
                cout << "输入库存: "; cin >> stock;
                books.emplace_back(id, title, author, stock);
                cout << "添加成功！\n";
                break;
            case 2:
                cout << "输入要删除的图书ID: "; cin >> id;
                books.erase(remove_if(books.begin(), books.end(), [&](Book& b) { return b.getId() == id; }), books.end());
                cout << "删除操作完成。\n";
                break;
            case 3:
                cout << "输入要修改的图书ID: "; cin >> id;
                {
                    Book* b = findBook(id);
                    if (b) {
                        cout << "输入新书名: "; cin >> title;
                        cout << "输入新作者: "; cin >> author;
                        cout << "输入新库存: "; cin >> stock;
                        b->setInfo(title, author, stock);
                        cout << "修改成功！\n";
                    }
                    else cout << "图书不存在！\n";
                }
                break;
            case 4:
                cout << "输入查询关键词(ID或书名): "; cin >> id;
                cout << left << setw(10) << "ID" << setw(20) << "书名" << setw(15) << "作者" << "库存/可借\n";
                for (const auto& b : books) {
                    if (b.getId().find(id) != string::npos || b.getTitle().find(id) != string::npos) {
                        cout << setw(10) << b.getId() << setw(20) << b.getTitle() << setw(15) << b.getAuthor()
                            << b.getStock() << "/" << b.getAvailable() << endl;
                    }
                }
                break;
            }
        }
    }

    void adminViewReport() {
        cout << "\n=== 系统统计报表 ===\n";
        cout << "总图书种类: " << books.size() << endl;
        int totalBooks = 0, borrowedBooks = 0;
        for (const auto& b : books) { totalBooks += b.getStock(); borrowedBooks += (b.getStock() - b.getAvailable()); }
        cout << "总藏书量: " << totalBooks << endl;
        cout << "已借出: " << borrowedBooks << endl;
        cout << "总用户数: " << users.size() << endl;

        cout << "\n--- 近期借阅记录 ---\n";
        int count = 0;
        for (auto it = records.rbegin(); it != records.rend() && count < 10; ++it) {
            cout << "记录ID:" << it->getBookId() << " | 借出日:" << it->getBorrowDate()
                << " | 状态:" << (it->getIsReturned() ? "已还" : "借出中") << endl;
            count++;
        }
        Utils::pauseConsole();
    }

    void adminManageUsers() {
        string id, name, pass, role;
        cout << "\n=== 创建新用户 ===\n";
        cout << "输入用户ID: "; cin >> id;
        if (users.find(id) != users.end()) { cout << "用户已存在！\n"; return; }
        cout << "输入姓名: "; cin >> name;
        cout << "输入密码: "; cin >> pass;
        cout << "输入角色 (admin/reader): "; cin >> role;

        if (role == "admin") users[id] = new Admin(id, name, pass);
        else users[id] = new Reader(id, name, pass);
        cout << "用户创建成功！\n";
    }

    // ================== 读者功能实现 ==================
    void readerBorrowReturn() {
        int choice; string bookId;
        while (true) {
            cout << "\n=== 借阅/归还 ===\n1. 借书\n2. 还书\n3. 我的借阅记录\n0. 返回\n选择: ";
            cin >> choice;
            if (choice == 0) break;

            switch (choice) {
            case 1:
                cout << "输入要借阅的图书ID: "; cin >> bookId;
                if (borrowBook(bookId, currentUser->getId())) cout << "借阅成功！\n";
                else cout << "借阅失败（图书不存在、无库存或已达借阅上限）。\n";
                break;
            case 2:
                cout << "输入要归还的图书ID: "; cin >> bookId;
                if (returnBook(bookId)) cout << "归还成功！\n";
                else cout << "归还失败（未找到您的该本在借图书记录）。\n";
                break;
            case 3:
                cout << "\n--- 我的借阅历史 ---\n";
                for (const auto& r : records) {
                    if (r.getReaderId() == currentUser->getId()) {
                        Book* b = findBook(r.getBookId());
                        string bName = b ? b->getTitle() : "未知图书";
                        cout << "图书: " << bName << " | 借于: " << r.getBorrowDate()
                            << " | 状态: " << (r.getIsReturned() ? "已归还" : "借阅中") << endl;
                    }
                }
                Utils::pauseConsole();
                break;
            }
        }
    }

    void readerSearchBooks() {
        string keyword;
        cout << "输入搜索关键词: "; cin >> keyword;
        cout << left << setw(10) << "ID" << setw(20) << "书名" << setw(15) << "作者" << "状态\n";
        //left:左对齐,sew:给每一列预留固定的宽度（比如ID占10个字符宽，书名占20个）
        for (const auto& b : books) {
            if (b.getId().find(keyword) != string::npos || b.getTitle().find(keyword) != string::npos) {
                cout << setw(10) << b.getId() << setw(20) << b.getTitle() << setw(15) << b.getAuthor()
                    << (b.isAvailable() ? "可借" : "已借出") << endl;
            }
        }
        Utils::pauseConsole();//让屏幕停在这里
    }

    // 友元声明以便User子类调用私有方法
    friend void Reader::showMenu(LibrarySystem& sys);
    friend void Admin::showMenu(LibrarySystem& sys);
};

// ===================== 菜单实现 =====================

void Admin::showMenu(LibrarySystem& sys) {
    int choice;
    do {
        cout << "\n========== 管理员控制台 ==========\n";
        cout << "1. 图书信息管理 (增删改查)\n";
        cout << "2. 用户管理 (创建新用户)\n";
        cout << "3. 查看系统报表与统计\n";
        cout << "4. 手动保存数据\n";
        cout << "0. 退出登录\n";
        cout << "请选择操作: ";
        cin >> choice;

        switch (choice) {
        case 1: sys.adminManageBooks(); break;
        case 2: sys.adminManageUsers(); break;
        case 3: sys.adminViewReport(); break;
        case 4: sys.saveData(); cout << "数据已保存！\n"; break;
        case 0: cout << "正在退出...\n"; break;
        default: cout << "无效选项！\n";
        }
    } while (choice != 0);
}

void Reader::showMenu(LibrarySystem& sys) {
    int choice;
    do {
        cout << "\n========== 读者服务大厅 ==========\n";
        cout << "1. 查询图书\n";
        cout << "2. 借书 / 还书\n";
        cout << "3. 查看个人借阅记录\n";
        cout << "请选择操作: ";
        cin >> choice;

        switch (choice) {
        case 1:
            sys.readerSearchBooks();
            break;
        case 2:
            sys.readerBorrowReturn();
            break;
        case 3:
            sys.readerBorrowReturn(); // 复用借阅归还里的查看记录功能，或者单独实现
            break;
        case 0:
            cout << "正在退出登录...\n";
            break;
        default:
            cout << "无效选项！\n";
        }
    } while (choice != 0);
}

// ===================== 主函数 (Main Entry) =====================
int main() {
    LibrarySystem library;

    // 欢迎界面
    while (true) {
        cout << "\n**********************************************\n";
        cout << "*                                            *\n";
        cout << "*       欢迎使用 C++ 图书馆管理系统          *\n";
        cout << "*                                            *\n";
        cout << "**********************************************\n";
        cout << "1. 进入登录界面\n";
        cout << "2. 关于本系统\n";
        cout << "0. 退出程序\n";
        cout << "请选择: ";

        int mainChoice;
        if (!(cin >> mainChoice)) {
            Utils::clearInputBuffer();
            continue;
        }

        switch (mainChoice) {
        case 1:
            library.run(); // 进入系统核心运行循环
            break;
        case 2:
            cout << "\n--- 关于系统 ---\n";
            cout << "版本: v1.0.0\n";
            cout << "开发者: AI助手\n";
            cout << "说明: 本系统基于C++面向对象编程思想开发，\n支持管理员与读者双角色，具备数据持久化功能。\n";
            Utils::pauseConsole();
            break;
        case 0:
            cout << "感谢使用，再见！\n";
            return 0;
        default:
            cout << "输入错误，请重新选择。\n";
        }
    }
    return 0;
}
