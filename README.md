# Self-judging-traffic-lights
// ============================================================================
//  人流量自适应红绿灯 
//  ---------------------------------------------------------------------------
#include <iostream>
#include <string>
#include <vector>
#include <queue>
#include <random>
#include <thread>
#include <chrono>
#include <iomanip>
#include <algorithm>

#ifdef _WIN32
// windows.h 默认会定义 min / max 两个宏，它们会把 std::max、std::min 顶掉，
// 编译时报 "error C2589: "(":"::"右边的非法标记"。
// NOMINMAX 让 windows.h 不定义这两个宏；必须在 #include <windows.h> 之前写。
#ifndef NOMINMAX
#define NOMINMAX
#endif
#include <windows.h>
#endif

// ============================================================================
//  第 1 课：固定配时红绿灯
//  ---------------------------------------------------------------------------
//  学习目标
//    1. enum class：用"枚举"表示有限的几种状态
//    2. 状态机：核心是一个 LightColor 变量，只会在三个值之间跳
//    3. 循环 + 计时：每一"秒"做一次判断，时间到了就切换状态
//    4. 函数：把"打印灯""该亮多久"拆成小函数
//
//  这是"笨"红绿灯：不管有没有车、有没有人，时间都是死的。
// ============================================================================
namespace lesson1 {

enum class LightColor { Green, Yellow, Red };

std::string colorName(LightColor c) {
    switch (c) {
    case LightColor::Green:  return "绿灯";
    case LightColor::Yellow: return "黄灯";
    case LightColor::Red:    return "红灯";
    }
    return "未知";
}

// 画一个竖着的三色灯：● 亮，○ 灭
void printLight(LightColor current) {
    auto lamp = [current](LightColor me) {
        return (current == me) ? "●" : "○";
    };

    std::cout << "\n";
    std::cout << "      ╔═════════╗\n";
    std::cout << "      ║    " << lamp(LightColor::Red)    << "    ║\n";
    std::cout << "      ║    " << lamp(LightColor::Yellow) << "    ║\n";
    std::cout << "      ║    " << lamp(LightColor::Green)  << "    ║\n";
    std::cout << "      ╚═════════╝  当前：" << colorName(current) << "\n";
}

// 1 个仿真秒 = 0.12 真实秒，看起来快 8 倍多，不然演示一遍要一分多钟
const int REAL_MS_PER_SIM_SECOND = 120;

void tick() {
    std::this_thread::sleep_for(std::chrono::milliseconds(REAL_MS_PER_SIM_SECOND));
}

int durationOf(LightColor c) {
    switch (c) {
    case LightColor::Green:  return 20;
    case LightColor::Yellow: return 3;
    case LightColor::Red:    return 20;
    }
    return 0;
}

// 状态转移：绿 -> 黄 -> 红 -> 绿
LightColor nextColor(LightColor c) {
    switch (c) {
    case LightColor::Green:  return LightColor::Yellow;
    case LightColor::Yellow: return LightColor::Red;
    case LightColor::Red:    return LightColor::Green;
    }
    return LightColor::Red;
}

int run() {
    std::cout << "===== 第 1 课：固定配时红绿灯 =====\n";
    std::cout << "规则：绿灯 20 秒 -> 黄灯 3 秒 -> 红灯 20 秒，共 3 轮\n";

    LightColor state = LightColor::Green;
    int remaining    = durationOf(state);
    int totalTicks   = 0;
    const int TOTAL  = 3 * (20 + 3 + 20);

    printLight(state);

    while (totalTicks < TOTAL) {
        tick();
        ++totalTicks;
        --remaining;

        if (remaining == 0) {
            state     = nextColor(state);
            remaining = durationOf(state);
            printLight(state);
        } else {
            // \r 让光标回到行首，覆盖上一次的倒计时
            std::cout << "\r      " << colorName(state)
                      << " 还剩 " << remaining << " 秒     " << std::flush;
        }
    }

    std::cout << "\n\n===== 演示结束 =====\n";
    std::cout << "问题来了：一辆车都没有的时候，红灯也傻等 20 秒。\n";
    std::cout << "下一课：让绿灯时长由排队车辆数来决定。\n";
    return 0;
}

} // namespace lesson1


// ============================================================================
//  第 2 课：让绿灯时长"自己算" —— 自适应配时的核心思想
//  ---------------------------------------------------------------------------
//  学习目标
//    1. struct：把"一个路口的情况"打包成一组数据
//    2. 纯函数：进去是数据，出来是秒数，不打印、不等待（好测试、好复用）
//    3. if / else 与边界裁剪（clamp）：把结果限制在合理范围内
//
//  公式：
//      绿灯时长 = 基础时间 + 排队车辆数 × 每辆车需要的放行时间
//      修正：有行人 -> 延长；对面更堵 -> 让出时间
//      最后裁剪到 [MIN_GREEN, MAX_GREEN]
// ============================================================================
namespace lesson2 {

struct RoadFlow {
    std::string name;        // 方向名字，比如 "南-北"
    int carsWaiting;         // 排队等着的车（辆）
    int pedestriansWaiting;  // 等着过街的行人（人）
};

int decideGreenSeconds(const RoadFlow& myFlow, const RoadFlow& otherFlow) {
    const int MIN_GREEN = 8;   // 最短绿灯：再没人也得让已经起步的车走完
    const int MAX_GREEN = 35;  // 最长绿灯：不然对面要等到骂人
    const int BASE      = 8;
    const int PER_CAR   = 2;

    int green = BASE + myFlow.carsWaiting * PER_CAR;

    // 有人等着过街，多给一点
    if (myFlow.pedestriansWaiting > 0)  green += 3;
    if (myFlow.pedestriansWaiting >= 5) green += 4;

    // 对面也在排长队，就别让自己把时间吃光
    if (otherFlow.carsWaiting >= 5 &&
        otherFlow.carsWaiting > myFlow.carsWaiting * 2) {
        green -= 4;
    }

    if (green < MIN_GREEN) green = MIN_GREEN;
    if (green > MAX_GREEN) green = MAX_GREEN;
    return green;
}

std::string explain(const RoadFlow& myFlow, const RoadFlow& otherFlow, int green) {
    if (myFlow.carsWaiting == 0 && otherFlow.carsWaiting == 0) {
        return "两边都没车——给最短时间，赶紧换下一个方向";
    }
    if (green >= 30) {
        return "本方向排长队，按每车 2 秒算，接近上限，多放一会儿";
    }
    if (otherFlow.carsWaiting > myFlow.carsWaiting * 2 && otherFlow.carsWaiting >= 5) {
        return "对面明显更堵——本方向够用就行，把时间让给对面";
    }
    if (myFlow.pedestriansWaiting >= 5) {
        return "行人多，延长绿灯，一次多过几个人";
    }
    if (myFlow.carsWaiting <= 2) {
        return "车很少——短绿灯就够，减少空放";
    }
    return "按排队车辆数正常配时";
}

int run() {
    std::cout << "===== 第 2 课：自适应配时决策表 =====\n\n";

    struct Scenario {
        std::string title;
        RoadFlow    ns;
        RoadFlow    ew;
    };

    std::vector<Scenario> scenes = {
        { "深夜：两边都没车",
          { "南-北", 0, 0 },  { "东-西", 0, 0 } },

        { "早高峰：东西路是主干道，排了 18 辆",
          { "南-北", 2, 0 },  { "东-西", 18, 0 } },

        { "学校门口：南北有 6 个行人等着过街",
          { "南-北", 3, 6 },  { "东-西", 4, 0 } },

        { "单方向爆堵：南北排了 20 辆",
          { "南-北", 20, 0 }, { "东-西", 1, 0 } },

        { "普通时段：两边各排 5 辆",
          { "南-北", 5, 0 },  { "东-西", 5, 0 } },
    };

    for (std::size_t i = 0; i < scenes.size(); ++i) {
        const Scenario& s = scenes[i];

        int greenNS = decideGreenSeconds(s.ns, s.ew);
        int greenEW = decideGreenSeconds(s.ew, s.ns);

        std::cout << "[" << (i + 1) << "] " << s.title << "\n";
        std::cout << "    排队情况："
                  << s.ns.name << " " << s.ns.carsWaiting << " 辆（行人 "
                  << s.ns.pedestriansWaiting << " 人）  |  "
                  << s.ew.name << " " << s.ew.carsWaiting << " 辆（行人 "
                  << s.ew.pedestriansWaiting << " 人）\n";
        std::cout << "    决策：南北绿灯 " << greenNS << " 秒，东西绿灯 "
                  << greenEW << " 秒\n";
        std::cout << "    南北侧理由：" << explain(s.ns, s.ew, greenNS) << "\n";
        std::cout << "    东西侧理由：" << explain(s.ew, s.ns, greenEW) << "\n\n";
    }

    std::cout << "===== 小结 =====\n";
    std::cout << "1) 决策就是一个函数：进去是数据，出来是秒数。\n";
    std::cout << "2) PER_CAR、MIN_GREEN、MAX_GREEN 就是调参的地方，\n";
    std::cout << "   改动它们，观察上面 5 个场景的结果怎么变——这就是工程。\n";
    std::cout << "3) 下一课：把这些数字放进一个会跑的仿真里。\n";
    return 0;
}

} // namespace lesson2


// ============================================================================
//  第 3 课：完整仿真 —— 让红绿灯自己看车流做决定
//  ---------------------------------------------------------------------------
//  学习目标
//    1. struct：把"一条路的排队情况"打包成 Road
//    2. 队列 queue：先进先出，正好对应"先来的车先走"
//    3. 用"相位(Phase)"建模十字路口：
//         南北绿 -> 南北黄 -> 全红 -> 东西绿 -> 东西黄 -> 全红
//    4. 自适应：按排队算时长；自己空了对面堵就提前切换；有行人也延一点
//    5. 统计：平均等待、最大排队、通行量、空放时间
//
//  约定：1 tick = 1 仿真秒；绿灯每秒最多放走 0.5 辆（约 1800 辆/小时）
// ============================================================================
namespace lesson3 {

const int    TOTAL_SECONDS    = 180;
const int    MIN_GREEN        = 8;
const int    MAX_GREEN        = 35;
const int    YELLOW_TIME      = 3;
const int    ALL_RED_TIME     = 2;
const double CARS_PER_SECOND  = 0.5;
const int    REAL_MS_PER_TICK = 45;   // 每仿真秒的真实毫秒数

const int PROB_NS_ARRIVE  = 30;   // 南北：每秒来车概率 %
const int PROB_EW_ARRIVE  = 50;   // 东西（主干道）：%
const int PROB_PED_ARRIVE = 12;   // 每秒来行人的概率 %

// ---- 一个方向的车道 ----
struct Road {
    std::string     name;
    std::queue<int> cars;          // 存每辆车的到达时刻，用于算等待
    int             passed    = 0;
    int             maxQueue  = 0;
    long long       totalWait = 0;

    explicit Road(const std::string& n) : name(n) {}

    void carArrives(int now) {
        cars.push(now);
        maxQueue = std::max(maxQueue, (int)cars.size());
    }

    int queueLength() const { return (int)cars.size(); }

    // accumulator 是"攒够 1 辆就放"的余量，实现 0.5 辆/秒
    int discharge(int now, double& accumulator) {
        accumulator += CARS_PER_SECOND;
        int released = 0;
        while (accumulator >= 1.0 && !cars.empty()) {
            int arrivedAt = cars.front();
            cars.pop();
            totalWait += (now - arrivedAt);
            ++passed;
            ++released;
            accumulator -= 1.0;
        }
        return released;
    }
};

// ---- 等过街的行人 ----
struct Pedestrians {
    std::queue<int> waiting;
    int             crossed   = 0;
    long long       totalWait = 0;

    void arrive(int now) { waiting.push(now); }

    int release(int now) {          // 跟南北绿灯同向过街，每秒最多 2 人
        int n = 0;
        while (n < 2 && !waiting.empty()) {
            totalWait += (now - waiting.front());
            waiting.pop();
            ++crossed;
            ++n;
        }
        return n;
    }

    int count() const { return (int)waiting.size(); }
};

// ---- 相位 ----
enum class Phase {
    NS_GREEN, NS_YELLOW, ALL_RED_1,
    EW_GREEN, EW_YELLOW, ALL_RED_2
};

std::string phaseName(Phase p) {
    switch (p) {
    case Phase::NS_GREEN:  return "南北·绿";
    case Phase::NS_YELLOW: return "南北·黄";
    case Phase::ALL_RED_1: return "全红";
    case Phase::EW_GREEN:  return "东西·绿";
    case Phase::EW_YELLOW: return "东西·黄";
    case Phase::ALL_RED_2: return "全红";
    }
    return "?";
}

// 这一轮绿灯给多少秒
int decideGreenSeconds(int myCars, int otherCars, int pedsWaiting) {
    const int BASE    = 8;
    const int PER_CAR = 2;

    int green = BASE + myCars * PER_CAR;

    if (pedsWaiting > 0)  green += 3;
    if (pedsWaiting >= 5) green += 4;
    if (otherCars >= 5 && otherCars > myCars * 2) green -= 4;

    green = std::max(green, MIN_GREEN);
    green = std::min(green, MAX_GREEN);
    return green;
}

// 自己空了、对面堵着 -> 没必要继续绿
bool shouldEndGreenEarly(int myCars, int otherCars, int elapsed) {
    return myCars == 0 && otherCars >= 4 && elapsed >= MIN_GREEN;
}

std::string queueBar(int n, int width = 8) {
    std::string s;
    for (int i = 0; i < width; ++i) s += (i < n ? "■" : "□");
    if (n > width) s += "+" + std::to_string(n - width);
    return s;
}

void printStatus(int t, const Road& ns, const Road& ew, Phase phase,
                 int remaining, const Pedestrians& peds) {
    std::cout << "t=" << std::setw(3) << t << " | "
              << "南北 " << queueBar(ns.queueLength()) << std::setw(3) << ns.queueLength() << " 辆"
              << " | 东西 " << queueBar(ew.queueLength()) << std::setw(3) << ew.queueLength() << " 辆"
              << " | " << phaseName(phase) << " 剩 " << std::setw(2) << remaining << "s"
              << " | 行人等 " << peds.count() << "\n";
}

int run() {
    // 固定种子：每次结果一样，方便对照。想每次不同改为 random_device
    std::mt19937 rng(2026);
    std::uniform_int_distribution<int> chance(1, 100);

    Road ns{"南-北"};
    Road ew{"东-西"};
    Pedestrians peds;

    Phase phase     = Phase::NS_GREEN;
    int   planned   = decideGreenSeconds(ns.queueLength(), ew.queueLength(), peds.count());
    int   remaining = planned;

    double accNS = 0.0;
    double accEW = 0.0;
    int    wastedGreen = 0;

    std::cout << "===== 第 3 课：自适应红绿灯仿真 =====\n";
    std::cout << "仿真 " << TOTAL_SECONDS << " 秒，南北来车率 " << PROB_NS_ARRIVE
              << "%，东西来车率 " << PROB_EW_ARRIVE << "%\n\n";

    for (int t = 1; t <= TOTAL_SECONDS; ++t) {

        // (1) 随机生成到达
        if (chance(rng) <= PROB_NS_ARRIVE) ns.carArrives(t);
        if (chance(rng) <= PROB_EW_ARRIVE) ew.carArrives(t);
        if (chance(rng) <= PROB_PED_ARRIVE) peds.arrive(t);

        // (2) 当前相位干活
        switch (phase) {
        case Phase::NS_GREEN:
            if (ns.queueLength() == 0) ++wastedGreen;
            ns.discharge(t, accNS);
            peds.release(t);
            break;
        case Phase::EW_GREEN:
            if (ew.queueLength() == 0) ++wastedGreen;
            ew.discharge(t, accEW);
            break;
        default:
            break;   // 黄灯、全红：谁都不许走
        }

        --remaining;

        // (3) 判断该不该换相位
        bool finished = (remaining <= 0);

        if (phase == Phase::NS_GREEN && !finished) {
            int elapsed = planned - remaining;
            if (shouldEndGreenEarly(ns.queueLength(), ew.queueLength(), elapsed))
                finished = true;
        }
        if (phase == Phase::EW_GREEN && !finished) {
            int elapsed = planned - remaining;
            if (shouldEndGreenEarly(ew.queueLength(), ns.queueLength(), elapsed))
                finished = true;
        }

        printStatus(t, ns, ew, phase, remaining, peds);

        if (finished) {
            // (4) 相位转移
            switch (phase) {
            case Phase::NS_GREEN:
                phase = Phase::NS_YELLOW;  remaining = YELLOW_TIME;  break;
            case Phase::NS_YELLOW:
                phase = Phase::ALL_RED_1;  remaining = ALL_RED_TIME; break;
            case Phase::ALL_RED_1:
                phase     = Phase::EW_GREEN;
                planned   = decideGreenSeconds(ew.queueLength(), ns.queueLength(), 0);
                remaining = planned;
                break;
            case Phase::EW_GREEN:
                phase = Phase::EW_YELLOW;  remaining = YELLOW_TIME;  break;
            case Phase::EW_YELLOW:
                phase = Phase::ALL_RED_2;  remaining = ALL_RED_TIME; break;
            case Phase::ALL_RED_2:
                phase     = Phase::NS_GREEN;
                planned   = decideGreenSeconds(ns.queueLength(), ew.queueLength(), peds.count());
                remaining = planned;
                break;
            }
        }
    }

    // (5) 统计
    std::cout << "\n===== 仿真结束，统计结果 =====\n";
    auto report = [](const Road& r) {
        double avg = r.passed > 0 ? (double)r.totalWait / r.passed : 0.0;
        std::cout << "  " << r.name << "：通过 " << r.passed << " 辆"
                  << "，平均等待 " << std::fixed << std::setprecision(1) << avg << " 秒"
                  << "，最长排队 " << r.maxQueue << " 辆\n";
    };
    report(ns);
    report(ew);

    double pedAvg = peds.crossed > 0 ? (double)peds.totalWait / peds.crossed : 0.0;
    std::cout << "  行人：通过 " << peds.crossed << " 人"
              << "，平均等待 " << std::fixed << std::setprecision(1) << pedAvg << " 秒\n";
    std::cout << "  绿灯空放（亮着却没车）：" << wastedGreen << " 秒\n";

    std::cout << "\n===== 可以动手试试 =====\n";
    std::cout << "  1. 把 PROB_EW_ARRIVE 改成 80，看东西向平均等待怎么变。\n";
    std::cout << "  2. 把 PER_CAR 从 2 改成 3，绿灯会变长，空放时间也会变多。\n";
    std::cout << "  3. 把 shouldEndGreenEarly 里的 4 改成 10，体会更激进的提前切换。\n";
    std::cout << "  4. 加一个变量记录每种相位各占多少秒，看看时间都花在哪。\n";
    return 0;
}

} // namespace lesson3


// ============================================================================
//  入口：整个文件只有这一个 main()
// ============================================================================
static void printMenu() {
    std::cout << "\n+------------------------------------------+\n";
    std::cout << "|   人流量自适应红绿灯 · 三课合一           |\n";
    std::cout << "+------------------------------------------+\n";
    std::cout << "|   1  第 1 课：固定配时（先看这个）        |\n";
    std::cout << "|   2  第 2 课：自适应决策表                |\n";
    std::cout << "|   3  第 3 课：完整仿真                    |\n";
    std::cout << "|   0  退出                                 |\n";
    std::cout << "+------------------------------------------+\n";
    std::cout << "请选择 > " << std::flush;
}

int main() {
#ifdef _WIN32
    // 【这里为什么不能写死成 CP_UTF8】
    // 源码是 UTF-8，但"源码编码"和"程序里存的字节"是两件事：
    //   * MSVC 不加 /utf-8 时（就是你现在的 one.vcxproj），
    //     字符串字面量会被转成系统 ANSI 编码 —— 中文 Windows 上是 GBK(936)。
    //   * g++ / MinGW（Dev-C++）则原样保留 UTF-8 字节。
    // 所以两边必须设不同的控制台代码页，写死任何一个都会有一边显示乱码
    // （"鍒濆鍖�" 或 "?????" 就是这个原因，不是逻辑错）。
#ifdef __GNUC__
    SetConsoleOutputCP(CP_UTF8);    // g++：字节是 UTF-8，控制台也按 UTF-8
#else
    SetConsoleOutputCP(GetACP());   // MSVC：字节是系统 ANSI(GBK)，让控制台按它显示
#endif
    // 进阶：以后若在项目属性里加了 /utf-8，把上面改成 CP_UTF8 即可。
#endif

    std::string line;
    while (true) {
        printMenu();
        if (!std::getline(std::cin, line)) break;   // 输入流结束（比如管道）
        if (line.empty()) continue;

        std::cout << "\n";
        switch (line[0]) {
        case '1': lesson1::run(); break;
        case '2': lesson2::run(); break;
        case '3': lesson3::run(); break;
        case '0':
            std::cout << "再见！\n";
            return 0;
        default:
            std::cout << "输入无效，只能输入 0 / 1 / 2 / 3。\n";
            continue;
        }

        std::cout << "\n---- 按回车返回菜单 ----" << std::flush;
        if (!std::getline(std::cin, line)) break;
    }
    return 0;
}
