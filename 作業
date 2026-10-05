
#include <windows.h>

#include <cstdint>
#include <ctime>
#include <fstream>
#include <iomanip>
#include <sstream>
#include <string>
#include <vector>

using namespace std;

struct Entry {
    wstring note;
    double income;
    double expense;
};

struct CurrentDate {
    wstring month;
    wstring fullDate;
};

enum ControlId {
    ID_USERNAME = 101,
    ID_PASSWORD = 102,
    ID_LOGIN = 103,
    ID_ADD = 201,
    ID_ENTRIES = 202,
    ID_SUMMARY = 203,
    ID_RESET = 204,
    ID_LOGOUT = 205,
    ID_NOTE = 301,
    ID_INCOME = 302,
    ID_EXPENSE = 303,
    ID_SAVE = 304,
    ID_BACK = 305,
    ID_DELETE = 306,
    ID_LIST = 401
};

HINSTANCE g_instance;
HWND g_window;
HFONT g_titleFont;
HFONT g_textFont;
vector<HWND> g_controls;
vector<Entry> g_entries;
CurrentDate g_currentDate;
wstring g_currentUser;
bool g_isAdmin = false;

string currentDataFile() {
    return "accounting_" + string(g_currentUser.begin(), g_currentUser.end()) + ".dat";
}

wstring formatMoney(double value) {
    wostringstream output;
    output << fixed << setprecision(2) << value;
    return output.str();
}

wstring formatCount(size_t value) {
    wostringstream output;
    output << value;
    return output.str();
}

double totalIncome() {
    double total = 0;
    for (const Entry& entry : g_entries) {
        total += entry.income;
    }
    return total;
}

double totalExpense() {
    double total = 0;
    for (const Entry& entry : g_entries) {
        total += entry.expense;
    }
    return total;
}

CurrentDate getCurrentDate() {
    time_t now = time(nullptr);
    tm localTime{};

#ifdef _WIN32
    localtime_s(&localTime, &now);
#else
    localtime_r(&now, &localTime);
#endif

    const wchar_t* weekdays[] = {
        L"星期日", L"星期一", L"星期二", L"星期三",
        L"星期四", L"星期五", L"星期六"
    };

    const int year = localTime.tm_year + 1900;
    const int month = localTime.tm_mon + 1;
    const int day = localTime.tm_mday;

    wostringstream monthStream;
    monthStream << year << L"-" << setw(2) << setfill(L'0') << month;

    wostringstream dateStream;
    dateStream << year << L"-" << setw(2) << setfill(L'0') << month
               << L"-" << setw(2) << setfill(L'0') << day
               << L" " << weekdays[localTime.tm_wday];

    return {monthStream.str(), dateStream.str()};
}

wstring trim(const wstring& text) {
    const wstring spaces = L" \t\r\n";
    const size_t start = text.find_first_not_of(spaces);
    if (start == wstring::npos) {
        return L"";
    }

    const size_t end = text.find_last_not_of(spaces);
    return text.substr(start, end - start + 1);
}

wstring getText(HWND control) {
    int length = GetWindowTextLengthW(control);
    wstring value(length + 1, L'\0');
    GetWindowTextW(control, value.data(), length + 1);
    value.resize(length);
    return trim(value);
}

bool parseMoney(const wstring& text, double& value) {
    wistringstream input(text);
    wchar_t extra;
    return input >> value && !(input >> extra) && value >= 0;
}

bool loadEntries() {
    g_entries.clear();

    if (g_currentUser.empty()) {
        return false;
    }

    ifstream input(currentDataFile(), ios::binary);
    if (!input) {
        return false;
    }

    char magic[8]{};
    uint32_t version = 0;
    uint32_t count = 0;

    input.read(magic, sizeof(magic));
    input.read(reinterpret_cast<char*>(&version), sizeof(version));
    input.read(reinterpret_cast<char*>(&count), sizeof(count));

    if (!input || string(magic, sizeof(magic)) != "ACCTDAT1" || version != 1) {
        g_entries.clear();
        return false;
    }

    for (uint32_t i = 0; i < count; ++i) {
        uint32_t noteLength = 0;
        Entry entry{};

        input.read(reinterpret_cast<char*>(&noteLength), sizeof(noteLength));
        entry.note.resize(noteLength);
        if (noteLength > 0) {
            input.read(reinterpret_cast<char*>(entry.note.data()),
                       static_cast<streamsize>(noteLength * sizeof(wchar_t)));
        }
        input.read(reinterpret_cast<char*>(&entry.income), sizeof(entry.income));
        input.read(reinterpret_cast<char*>(&entry.expense), sizeof(entry.expense));

        if (!input) {
            g_entries.clear();
            return false;
        }

        g_entries.push_back(entry);
    }

    return true;
}

void saveEntries() {
    if (g_currentUser.empty()) {
        return;
    }

    ofstream output(currentDataFile(), ios::binary | ios::trunc);
    if (!output) {
        MessageBoxW(g_window, L"資料檔案無法儲存，請確認程式所在資料夾可以寫入。", L"儲存失敗",
                    MB_OK | MB_ICONWARNING);
        return;
    }

    const char magic[8] = {'A', 'C', 'C', 'T', 'D', 'A', 'T', '1'};
    const uint32_t version = 1;
    const uint32_t count = static_cast<uint32_t>(g_entries.size());

    output.write(magic, sizeof(magic));
    output.write(reinterpret_cast<const char*>(&version), sizeof(version));
    output.write(reinterpret_cast<const char*>(&count), sizeof(count));

    for (const Entry& entry : g_entries) {
        const uint32_t noteLength = static_cast<uint32_t>(entry.note.size());
        output.write(reinterpret_cast<const char*>(&noteLength), sizeof(noteLength));
        if (noteLength > 0) {
            output.write(reinterpret_cast<const char*>(entry.note.data()),
                         static_cast<streamsize>(noteLength * sizeof(wchar_t)));
        }
        output.write(reinterpret_cast<const char*>(&entry.income), sizeof(entry.income));
        output.write(reinterpret_cast<const char*>(&entry.expense), sizeof(entry.expense));
    }
}

void clearControls() {
    for (HWND control : g_controls) {
        DestroyWindow(control);
    }
    g_controls.clear();
}

HWND addControl(const wchar_t* className, const wstring& text, DWORD style,
                int x, int y, int width, int height, int id = 0) {
    HWND control = CreateWindowExW(
        0, className, text.c_str(), WS_CHILD | WS_VISIBLE | style,
        x, y, width, height, g_window, reinterpret_cast<HMENU>(id),
        g_instance, nullptr);

    SendMessageW(control, WM_SETFONT, reinterpret_cast<WPARAM>(g_textFont), TRUE);
    g_controls.push_back(control);
    return control;
}

HWND addLabel(const wstring& text, int x, int y, int width, int height) {
    return addControl(L"STATIC", text, 0, x, y, width, height);
}

HWND addButton(const wstring& text, int x, int y, int width, int height, int id) {
    return addControl(L"BUTTON", text, BS_PUSHBUTTON, x, y, width, height, id);
}

HWND addEdit(const wstring& text, int x, int y, int width, int height, int id,
             bool password = false) {
    DWORD style = WS_BORDER | ES_AUTOHSCROLL;
    if (password) {
        style |= ES_PASSWORD;
    }
    return addControl(L"EDIT", text, style, x, y, width, height, id);
}

void addTitle(const wstring& text) {
    HWND title = addLabel(text, 30, 24, 520, 34);
    SendMessageW(title, WM_SETFONT, reinterpret_cast<WPARAM>(g_titleFont), TRUE);
}

void showLogin();
void showDashboard();
void showAddEntry();
void showEntries();

void showLogin() {
    clearControls();
    SetWindowTextW(g_window, L"記帳系統 - 登入");

    addTitle(L"記帳系統");
    addLabel(L"示範帳號：admin / 1234、user / 0000", 30, 70, 460, 24);
    addLabel(L"帳號", 60, 120, 100, 24);
    addEdit(L"", 170, 118, 250, 28, ID_USERNAME);
    addLabel(L"密碼", 60, 165, 100, 24);
    addEdit(L"", 170, 163, 250, 28, ID_PASSWORD, true);
    addButton(L"登入", 170, 215, 120, 34, ID_LOGIN);
}

void showDashboard() {
    clearControls();
    SetWindowTextW(g_window, L"記帳系統 - 主畫面");

    const double income = totalIncome();
    const double expense = totalExpense();
    const double balance = income - expense;

    addTitle(L"每月記帳系統");
    addLabel(L"今天日期：" + g_currentDate.fullDate, 30, 70, 520, 24);
    addLabel(L"記帳月份：" + g_currentDate.month, 30, 98, 240, 24);
    addLabel(L"使用者：" + g_currentUser + (g_isAdmin ? L"（管理者）" : L"（一般使用者）"),
             290, 98, 300, 24);

    addLabel(L"資料筆數：" + formatCount(g_entries.size()), 55, 150, 220, 24);
    addLabel(L"總收入：" + formatMoney(income), 55, 185, 240, 24);
    addLabel(L"總支出：" + formatMoney(expense), 55, 220, 240, 24);
    addLabel(L"結餘：" + formatMoney(balance), 55, 255, 240, 24);

    addButton(L"新增記帳", 350, 145, 150, 34, ID_ADD);
    addButton(L"查看明細", 350, 190, 150, 34, ID_ENTRIES);
    addButton(L"月結摘要", 350, 235, 150, 34, ID_SUMMARY);

    addButton(L"清空紀錄", 350, 280, 150, 34, ID_RESET);
    addButton(L"登出", 350, 325, 150, 34, ID_LOGOUT);
}

void showAddEntry() {
    clearControls();
    SetWindowTextW(g_window, L"記帳系統 - 新增記帳");

    addTitle(L"新增記帳資料");
    addLabel(L"備註", 55, 95, 100, 24);
    addEdit(L"", 160, 92, 330, 28, ID_NOTE);
    addLabel(L"收入", 55, 145, 100, 24);
    addEdit(L"", 160, 142, 180, 28, ID_INCOME);
    addLabel(L"支出", 55, 195, 100, 24);
    addEdit(L"", 160, 192, 180, 28, ID_EXPENSE);

    addButton(L"儲存", 160, 250, 115, 34, ID_SAVE);
    addButton(L"返回", 290, 250, 115, 34, ID_BACK);
}

void showEntries() {
    clearControls();
    SetWindowTextW(g_window, L"記帳系統 - 明細");

    addTitle(L"記帳明細");

    HWND list = addControl(L"LISTBOX", L"", WS_BORDER | WS_VSCROLL | LBS_NOTIFY,
                           35, 80, 510, 250, ID_LIST);

    if (g_entries.empty()) {
        SendMessageW(list, LB_ADDSTRING, 0, reinterpret_cast<LPARAM>(L"目前沒有記帳資料。"));
    } else {
        for (size_t i = 0; i < g_entries.size(); ++i) {
            const Entry& entry = g_entries[i];
            wostringstream row;
            row << i + 1 << L". " << entry.note
                << L"｜收入：" << formatMoney(entry.income)
                << L"｜支出：" << formatMoney(entry.expense)
                << L"｜差額：" << formatMoney(entry.income - entry.expense);
            SendMessageW(list, LB_ADDSTRING, 0, reinterpret_cast<LPARAM>(row.str().c_str()));
        }
    }

    addButton(L"返回", 90, 350, 120, 34, ID_BACK);
    addButton(L"刪除選取", 230, 350, 120, 34, ID_DELETE);
    addButton(L"清空紀錄", 370, 350, 120, 34, ID_RESET);
}

void showSummaryMessage() {
    wostringstream message;
    message << L"記帳月份：" << g_currentDate.month << L"\n"
            << L"今天日期：" << g_currentDate.fullDate << L"\n\n"
            << L"總收入：" << formatMoney(totalIncome()) << L"\n"
            << L"總支出：" << formatMoney(totalExpense()) << L"\n"
            << L"結餘：" << formatMoney(totalIncome() - totalExpense());

    MessageBoxW(g_window, message.str().c_str(), L"月結摘要", MB_OK | MB_ICONINFORMATION);
}

void handleLogin() {
    wstring username = getText(GetDlgItem(g_window, ID_USERNAME));
    wstring password = getText(GetDlgItem(g_window, ID_PASSWORD));

    if (username == L"admin" && password == L"1234") {
        g_currentUser = username;
        g_isAdmin = true;
        loadEntries();
        showDashboard();
        return;
    }

    if (username == L"user" && password == L"0000") {
        g_currentUser = username;
        g_isAdmin = false;
        loadEntries();
        showDashboard();
        return;
    }

    MessageBoxW(g_window, L"帳號或密碼錯誤。", L"登入失敗", MB_OK | MB_ICONWARNING);
}

void handleSaveEntry() {
    Entry entry;
    entry.note = getText(GetDlgItem(g_window, ID_NOTE));
    const wstring incomeText = getText(GetDlgItem(g_window, ID_INCOME));
    const wstring expenseText = getText(GetDlgItem(g_window, ID_EXPENSE));

    if (entry.note.empty()) {
        MessageBoxW(g_window, L"請輸入備註。", L"缺少備註", MB_OK | MB_ICONWARNING);
        return;
    }

    if (!parseMoney(incomeText, entry.income)) {
        MessageBoxW(g_window, L"收入必須是大於或等於 0 的數字。", L"收入格式錯誤",
                    MB_OK | MB_ICONWARNING);
        return;
    }

    if (!parseMoney(expenseText, entry.expense)) {
        MessageBoxW(g_window, L"支出必須是大於或等於 0 的數字。", L"支出格式錯誤",
                    MB_OK | MB_ICONWARNING);
        return;
    }

    g_entries.push_back(entry);
    saveEntries();
    MessageBoxW(g_window, L"資料已儲存。", L"儲存成功", MB_OK | MB_ICONINFORMATION);
    showDashboard();
}

void handleClearEntries() {
    if (g_entries.empty()) {
        MessageBoxW(g_window, L"目前沒有紀錄可以清空。", L"清空紀錄", MB_OK | MB_ICONINFORMATION);
        return;
    }

    if (MessageBoxW(g_window, L"確定要清空所有記帳紀錄嗎？", L"確認清空",
                    MB_YESNO | MB_ICONQUESTION) == IDYES) {
        g_entries.clear();
        saveEntries();
        MessageBoxW(g_window, L"紀錄已清空。", L"清空完成", MB_OK | MB_ICONINFORMATION);
        showDashboard();
    }
}

void handleDeleteSelectedEntry() {
    HWND list = GetDlgItem(g_window, ID_LIST);
    if (!list || g_entries.empty()) {
        MessageBoxW(g_window, L"目前沒有紀錄可以刪除。", L"刪除紀錄", MB_OK | MB_ICONINFORMATION);
        return;
    }

    const LRESULT selectedIndex = SendMessageW(list, LB_GETCURSEL, 0, 0);
    if (selectedIndex == LB_ERR) {
        MessageBoxW(g_window, L"請先在明細列表選擇一筆紀錄。", L"尚未選取",
                    MB_OK | MB_ICONWARNING);
        return;
    }

    const size_t index = static_cast<size_t>(selectedIndex);
    if (index >= g_entries.size()) {
        MessageBoxW(g_window, L"選取的項目不是有效紀錄。", L"無法刪除", MB_OK | MB_ICONWARNING);
        return;
    }

    if (MessageBoxW(g_window, L"確定要刪除這筆紀錄嗎？", L"確認刪除",
                    MB_YESNO | MB_ICONQUESTION) == IDYES) {
        g_entries.erase(g_entries.begin() + index);
        saveEntries();
        MessageBoxW(g_window, L"紀錄已刪除。", L"刪除完成", MB_OK | MB_ICONINFORMATION);
        showEntries();
    }
}

LRESULT CALLBACK windowProcedure(HWND hwnd, UINT message, WPARAM wParam, LPARAM lParam) {
    switch (message) {
    case WM_COMMAND:
        switch (LOWORD(wParam)) {
        case ID_LOGIN:
            handleLogin();
            break;
        case ID_ADD:
            showAddEntry();
            break;
        case ID_ENTRIES:
            showEntries();
            break;
        case ID_SUMMARY:
            showSummaryMessage();
            break;
        case ID_RESET:
            handleClearEntries();
            break;
        case ID_LOGOUT:
            saveEntries();
            g_currentUser.clear();
            g_isAdmin = false;
            g_entries.clear();
            showLogin();
            break;
        case ID_SAVE:
            handleSaveEntry();
            break;
        case ID_DELETE:
            handleDeleteSelectedEntry();
            break;
        case ID_BACK:
            showDashboard();
            break;
        }
        return 0;

    case WM_DESTROY:
        saveEntries();
        DeleteObject(g_titleFont);
        DeleteObject(g_textFont);
        PostQuitMessage(0);
        return 0;
    }

    return DefWindowProcW(hwnd, message, wParam, lParam);
}

int main() {
    g_instance = GetModuleHandleW(nullptr);
    g_currentDate = getCurrentDate();

    g_titleFont = CreateFontW(26, 0, 0, 0, FW_BOLD, FALSE, FALSE, FALSE,
                              DEFAULT_CHARSET, OUT_DEFAULT_PRECIS, CLIP_DEFAULT_PRECIS,
                              DEFAULT_QUALITY, DEFAULT_PITCH | FF_SWISS, L"Microsoft JhengHei UI");
    g_textFont = CreateFontW(17, 0, 0, 0, FW_NORMAL, FALSE, FALSE, FALSE,
                             DEFAULT_CHARSET, OUT_DEFAULT_PRECIS, CLIP_DEFAULT_PRECIS,
                             DEFAULT_QUALITY, DEFAULT_PITCH | FF_SWISS, L"Microsoft JhengHei UI");

    WNDCLASSW windowClass{};
    windowClass.lpfnWndProc = windowProcedure;
    windowClass.hInstance = g_instance;
    windowClass.lpszClassName = L"ChineseAccountingSystemWindow";
    windowClass.hbrBackground = reinterpret_cast<HBRUSH>(COLOR_WINDOW + 1);
    windowClass.hCursor = LoadCursor(nullptr, IDC_ARROW);

    RegisterClassW(&windowClass);

    g_window = CreateWindowExW(
        0, L"ChineseAccountingSystemWindow", L"記帳系統",
        WS_OVERLAPPED | WS_CAPTION | WS_SYSMENU | WS_MINIMIZEBOX,
        CW_USEDEFAULT, CW_USEDEFAULT, 600, 460,
        nullptr, nullptr, g_instance, nullptr);

    if (!g_window) {
        return 1;
    }

    showLogin();
    ShowWindow(g_window, SW_SHOW);
    UpdateWindow(g_window);

    MSG message{};
    while (GetMessageW(&message, nullptr, 0, 0)) {
        TranslateMessage(&message);
        DispatchMessageW(&message);
    }

    return 0;
}
