<script lang="ts">
    import { goto } from '$app/navigation';
    import { getContext, onMount } from 'svelte';
    import { config, settings, user, models } from '$lib/stores';
    
    const i18n = getContext('i18n');

    // 로그아웃 함수
    const handleLogout = async () => {
        localStorage.removeItem('token');
        localStorage.removeItem('selectedPipeline');
        localStorage.removeItem('selectedModels');
        user.set(undefined);
        await goto('/auth');
    };

    onMount(async () => {
        if (!$user || !['user', 'admin'].includes($user.role)) {
            await goto('/auth');
            return;
        }
    });

    interface Pipeline {
        id: string;
        name: string;
        description: string;
        icon: string;
        modelId?: string;
    }

    // 파이프라인 목록
    let pipelines: Pipeline[] = [
        { 
            id: 'Azure OpenAI Pipeline', 
            name: "Azure OpenAI", 
            description: "Azure OpenAI 서비스를 이용한 대화",
            icon: "💬",
            modelId: "azure_openai_pipeline"
        },
        { 
            id: 'Data Analysis Pipeline', 
            name: "데이터 분석", 
            description: "데이터 분석을 위한 파이프라인",
            icon: "📊",
            modelId: "data_analysis_agent"
        },
        { 
            id: 'Simple Plot Pipeline', 
            name: "간단한 그래프 생성", 
            description: "간단한 그래프 생성을 위한 파이프라인",
            icon: "💻",
            modelId: "data_visualization_pipeline"
        }
    ];

    const handlePipelineSelect = async (pipeline: Pipeline) => {
        try {
            // 파이프라인 정보 저장
            localStorage.setItem('selectedPipeline', JSON.stringify(pipeline));
            
            // 선택된 모델 저장
            if (pipeline.modelId) {
                // 모델 목록에서 해당 ID를 가진 모델 찾기
                const modelMatch = $models.find(model => model.id === pipeline.modelId);
                if (modelMatch) {
                    const selectedModels = [modelMatch.id];
                    console.log('selectedModels', selectedModels);
                    
                    // localStorage에 저장
                    localStorage.setItem('selectedModels', JSON.stringify(selectedModels));
                    
                    // 사용자 설정에도 저장
                    if ($settings) {
                        settings.set({ ...$settings, models: selectedModels });
                    }
                }
            }

            // goto 함수로 먼저 시도
            try {
                console.log('goto /');
                await goto('/', { replaceState: true });
            } catch (error) {
                console.error('goto failed:', error);
                // goto가 실패하면 window.location 사용
                window.location.href = '/';
            }
        } catch (error) {
            console.error('파이프라인 선택 오류:', error);
        }
    };

    // 컴포넌트 마운트 시 사용 가능한 모델 목록 확인
    onMount(async () => {
        console.log('Available models:', $models);
    });
</script>

<div class="min-h-screen flex flex-col items-center justify-center bg-white dark:bg-gray-900 p-4">
    <!-- 로그아웃 버튼 -->
    <div class="absolute top-4 right-4">
        <button
            on:click={handleLogout}
            class="px-4 py-2 text-sm font-medium text-gray-700 dark:text-gray-200 
                   bg-white dark:bg-gray-800 rounded-lg border border-gray-200 dark:border-gray-700
                   hover:bg-gray-50 dark:hover:bg-gray-700 hover:text-gray-900 dark:hover:text-white
                   transition-colors duration-200"
        >
            로그아웃
        </button>
    </div>
    
    <div class="max-w-4xl w-full">
        <h1 class="text-2xl font-bold mb-2 text-center text-gray-900 dark:text-white">
            파이프라인 선택
        </h1>
        <p class="text-center mb-8 text-gray-600 dark:text-gray-400">
            사용하고자 하는 파이프라인을 선택하세요
        </p>
        
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
            {#each pipelines as pipeline}
                <button
                    on:click={() => handlePipelineSelect(pipeline)}
                    class="group flex flex-col p-6 bg-white dark:bg-gray-800 rounded-xl 
                           shadow-lg hover:shadow-xl transition-all duration-200 
                           border border-gray-100 dark:border-gray-700
                           hover:border-blue-500 dark:hover:border-blue-500"
                >
                    <div class="flex items-center gap-3 mb-3">
                        <span class="text-2xl">{pipeline.icon}</span>
                        <h3 class="text-lg font-medium text-gray-900 dark:text-white">
                            {pipeline.name}
                        </h3>
                    </div>
                    <p class="text-sm text-gray-500 dark:text-gray-400">
                        {pipeline.description}
                    </p>
                </button>
            {/each}
        </div>
    </div>
</div>

<style>
    :global(body) {
        overflow: hidden;
    }
</style> 