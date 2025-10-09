import { cache } from '../services/cacheService';
import { pollingService } from '../services/pollingService';
import { getInventoryItems } from '../services/inventoryService';
import { getTasks } from '../services/taskService';
import { getPresentUsers } from '../services/userService';

/**
 * Test suite to verify Firebase optimizations are working
 */
export class OptimizationTester {
  private testResults: { [key: string]: any } = {};

  /**
   * Run all optimization tests
   */
  async runAllTests(): Promise<void> {
    console.log('🧪 Starting Firebase Optimization Tests...\n');
    
    try {
      await this.testCacheSystem();
      await this.testQueryLimits();
      await this.testPollingSystem();
      await this.testQueryPerformance();
      
      this.printResults();
    } catch (error) {
      console.error('❌ Test suite failed:', error);
    }
  }

  /**
   * Test 1: Cache System
   */
  async testCacheSystem(): Promise<void> {
    console.log('1️⃣ Testing Cache System...');
    
    // Test cache set/get
    cache.set('test-key', { data: 'test-value' }, 1);
    const cached = cache.get('test-key');
    
    const cacheWorking = cached && (cached as any).data === 'test-value';
    this.testResults.cacheSystem = {
      working: cacheWorking,
      stats: cache.getStats()
    };
    
    console.log(cacheWorking ? '✅ Cache system working' : '❌ Cache system failed');
    console.log(`📊 Cache stats: ${this.testResults.cacheSystem.stats.size} entries`);
  }

  /**
   * Test 2: Query Limits
   */
  async testQueryLimits(): Promise<void> {
    console.log('\n2️⃣ Testing Query Limits...');
    
    const startTime = performance.now();
    
    // Test inventory query (should be limited to 500)
    const inventoryItems = await getInventoryItems();
    const inventoryTime = performance.now() - startTime;
    
    // Test tasks query (should be limited to 300)  
    const taskStartTime = performance.now();
    const tasks = await getTasks();
    const taskTime = performance.now() - taskStartTime;
    
    this.testResults.queryLimits = {
      inventoryCount: inventoryItems.length,
      inventoryTime: Math.round(inventoryTime),
      taskCount: tasks.length,
      taskTime: Math.round(taskTime),
      limitsWorking: inventoryItems.length <= 500 && tasks.length <= 300
    };
    
    console.log(this.testResults.queryLimits.limitsWorking ? '✅ Query limits working' : '❌ Query limits failed');
    console.log(`📦 Inventory items: ${inventoryItems.length}/500 limit`);
    console.log(`📋 Tasks: ${tasks.length}/300 limit`);
  }

  /**
   * Test 3: Polling System
   */
  async testPollingSystem(): Promise<void> {
    console.log('\n3️⃣ Testing Polling System...');
    
    let pollCount = 0;
    const testKey = 'test-polling';
    
    // Start a test poll
    pollingService.startPolling(
      testKey,
      async () => {
        pollCount++;
        return { timestamp: Date.now() };
      },
      (data) => {
        console.log(`📡 Poll received:`, data);
      },
      { interval: 2000, maxRetries: 1 }
    );
    
    // Wait for a few polls
    await new Promise(resolve => setTimeout(resolve, 5000));
    
    // Stop polling
    pollingService.stopPolling(testKey);
    
    const pollingStatus = pollingService.getPollingStatus();
    this.testResults.pollingSystem = {
      pollCount,
      pollingWorking: pollCount >= 2,
      activePolls: pollingStatus.activePolls
    };
    
    console.log(this.testResults.pollingSystem.pollingWorking ? '✅ Polling system working' : '❌ Polling system failed');
    console.log(`🔄 Polls executed: ${pollCount}`);
  }

  /**
   * Test 4: Query Performance
   */
  async testQueryPerformance(): Promise<void> {
    console.log('\n4️⃣ Testing Query Performance...');
    
    const tests = [
      { name: 'Present Users', fn: getPresentUsers },
      { name: 'Inventory Items', fn: getInventoryItems },
      { name: 'Tasks', fn: getTasks }
    ];
    
    const performanceResults: any[] = [];
    
    for (const test of tests) {
      const startTime = performance.now();
      
      try {
        const result = await test.fn();
        const duration = performance.now() - startTime;
        
        performanceResults.push({
          name: test.name,
          duration: Math.round(duration),
          count: Array.isArray(result) ? result.length : 1,
          success: true
        });
        
        console.log(`⚡ ${test.name}: ${Math.round(duration)}ms (${Array.isArray(result) ? result.length : 1} items)`);
      } catch (error) {
        performanceResults.push({
          name: test.name,
          duration: -1,
          count: 0,
          success: false,
          error: error
        });
        
        console.log(`❌ ${test.name}: Failed`);
      }
    }
    
    const avgDuration = performanceResults
      .filter(r => r.success)
      .reduce((sum, r) => sum + r.duration, 0) / performanceResults.filter(r => r.success).length;
    
    this.testResults.queryPerformance = {
      tests: performanceResults,
      averageDuration: Math.round(avgDuration),
      allPassed: performanceResults.every(r => r.success)
    };
  }

  /**
   * Print comprehensive test results
   */
  private printResults(): void {
    console.log('\n🎯 FIREBASE OPTIMIZATION TEST RESULTS');
    console.log('=====================================');
    
    // Overall status
    const allTestsPassed = 
      this.testResults.cacheSystem?.working &&
      this.testResults.queryLimits?.limitsWorking &&
      this.testResults.pollingSystem?.pollingWorking &&
      this.testResults.queryPerformance?.allPassed;
    
    console.log(allTestsPassed ? '🎉 ALL OPTIMIZATIONS WORKING!' : '⚠️  Some optimizations need attention');
    
    // Detailed results
    console.log('\n📊 DETAILED RESULTS:');
    console.log('Cache System:', this.testResults.cacheSystem?.working ? '✅' : '❌');
    console.log('Query Limits:', this.testResults.queryLimits?.limitsWorking ? '✅' : '❌');
    console.log('Polling System:', this.testResults.pollingSystem?.pollingWorking ? '✅' : '❌');
    console.log('Query Performance:', this.testResults.queryPerformance?.allPassed ? '✅' : '❌');
    
    // Performance summary
    if (this.testResults.queryPerformance?.averageDuration) {
      console.log(`\n⚡ Average Query Time: ${this.testResults.queryPerformance.averageDuration}ms`);
      console.log(this.testResults.queryPerformance.averageDuration < 1500 ? 
        '🚀 Excellent performance!' : 
        '⏳ Performance could be better'
      );
    }
    
    // Cache efficiency
    if (this.testResults.cacheSystem?.stats?.size > 0) {
      console.log(`\n🎯 Cache Efficiency: ${this.testResults.cacheSystem.stats.size} entries cached`);
    }
    
    console.log('\n📈 Next: Monitor Firebase Console for quota reduction!');
  }
}

// Export test runner
export const runOptimizationTests = async (): Promise<void> => {
  const tester = new OptimizationTester();
  await tester.runAllTests();
};

// Make available globally for console testing
(window as any).testFirebaseOptimizations = runOptimizationTests;