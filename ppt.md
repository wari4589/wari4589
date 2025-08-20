# EcoSort AI: 주요 코드 모음

## 1. AI 핵심 로직 (Genkit Flow) - `src/ai/flows/classify-waste.ts`

이 파일은 사용자가 입력한 폐기물에 대한 설명을 받아 카테고리와 처리 방법을 분류하는 AI의 핵심 로직입니다. Gemini 모델을 사용하여 대한민국 재활용 규정에 맞는 답변을 생성하도록 지시합니다.

```typescript
'use server';
/**
 * @fileOverview Classifies a waste item and provides disposal instructions.
 *
 * - classifyWaste - A function that classifies waste and provides instructions.
 * - ClassifyWasteInput - The input type for the classifyWaste function.
 * - ClassifyWasteOutput - The return type for the classifyWaste function.
 */

import {ai} from '@/ai/genkit';
import {z} from 'genkit';

const ClassifyWasteInputSchema = z.object({
  wasteItemDescription: z
    .string()
    .describe('The description of the waste item to classify.'),
});
export type ClassifyWasteInput = z.infer<typeof ClassifyWasteInputSchema>;

const ClassifyWasteOutputSchema = z.object({
  category: z.string().describe("The waste category of the item (e.g., paper, plastic, glass, general waste, etc.). For items like pizza boxes that can be contaminated, categorize it as 'paper' but provide instructions for both clean and contaminated states. If the category is unclear, provide the most likely categories."),
  disposalInstructions: z.string().describe('Step-by-step instructions for proper disposal, formatted as a numbered list with each step on a new line (using \\n).'),
});
export type ClassifyWasteOutput = z.infer<typeof ClassifyWasteOutputSchema>;

export async function classifyWaste(input: ClassifyWasteInput): Promise<ClassifyWasteOutput> {
  return classifyWasteFlow(input);
}

const prompt = ai.definePrompt({
  name: 'classifyWastePrompt',
  input: {schema: ClassifyWasteInputSchema},
  output: {schema: ClassifyWasteOutputSchema},
  prompt: `당신은 대한민국 폐기물 관리 및 재활용 전문가입니다.

제공된 설명을 바탕으로 폐기물 품목을 분류하고 명확한 처리 지침을 제공해야 합니다. 반드시 대한민국의 재활용 규정을 따라야 합니다. 답변은 한국어로 작성해 주세요.

폐기물 품목 설명: {{{wasteItemDescription}}}

품목의 주된 재질을 기반으로 카테고리를 지정해야 합니다. 예를 들어, 오염될 수 있는 피자 박스의 경우 주 재질인 '종이'로 분류하되, 처리 지침에는 깨끗한 부분과 오염된 부분을 어떻게 처리해야 하는지 명확하게 설명해야 합니다. 카테고리가 불분명할 경우, 가장 가능성이 높은 카테고리를 제시해주세요.

배출 방법은 번호가 매겨진 목록 형식으로 단계별로 제공해야 합니다. 각 단계는 반드시 줄바꿈(\\n)으로 구분되어야 합니다.`,
});

const classifyWasteFlow = ai.defineFlow(
  {
    name: 'classifyWasteFlow',
    inputSchema: ClassifyWasteInputSchema,
    outputSchema: ClassifyWasteOutputSchema,
  },
  async input => {
    const {output} = await prompt(input);
    return output!;
  }
);
```

## 2. 서버 액션 (Next.js Server Action) - `src/app/actions.ts`

이 파일은 프론트엔드(채팅 UI)와 AI 백엔드(Genkit Flow)를 연결하는 다리 역할을 합니다. 사용자의 채팅 기록을 받아, 첫 질문인지 후속 질문인지 판단하여 적절한 AI Flow를 호출합니다.

```typescript
'use server';

import { classifyWaste } from '@/ai/flows/classify-waste';
import { clarifyWasteDisposal } from '@/ai/flows/clarify-waste-disposal';

export interface ChatMessage {
  role: 'user' | 'ai';
  text: string;
}

export interface AIResponse {
  text: string;
  category?: string;
}

export async function handleWasteQuery(
  history: ChatMessage[]
): Promise<AIResponse> {
  const latestUserInput = history[history.length - 1].text;
  
  // 첫 질문인지, 아니면 후속 질문인지 판별
  // AI 메시지가 하나만 있고(초기 인사) 사용자가 질문하는 경우를 새로운 질문으로 간주
  const isFollowUp = history.filter(m => m.role === 'ai').length > 1;

  if (isFollowUp) {
    // 후속 질문 처리
    const conversationHistory = history.map(m => `${m.role}: ${m.text}`).join('\n\n');
    try {
      const response = await clarifyWasteDisposal({
        item: latestUserInput,
        conversationHistory,
      });
      return { text: response.disposalInstructions };
    } catch (error) {
      console.error('Error clarifying waste disposal:', error);
      return { text: "죄송합니다. 요청을 처리하는 데 문제가 발생했습니다. 질문을 다시 작성해 주시겠어요?" };
    }
  } else {
    // 첫 질문 또는 새로운 주제 처리
    try {
      const response = await classifyWaste({ wasteItemDescription: latestUserInput });
      return {
        text: response.disposalInstructions,
        category: response.category,
      };
    } catch (error) {
      console.error('Error classifying waste:', error);
      return { text: "죄송합니다. 해당 품목을 분류할 수 없습니다. 다른 방식으로 설명해 주세요." };
    }
  }
}
```

## 3. 채팅 UI 컴포넌트 - `src/components/chat-interface.tsx`

React의 `useState`, `useEffect` Hook을 사용하여 채팅 상태를 관리하고, 사용자의 입력을 받아 서버 액션(`handleWasteQuery`)을 호출하는 핵심 UI 컴포넌트입니다.

```typescript
"use client";

import { useState, useRef, useEffect, type FormEvent } from 'react';
import { SendHorizontal, Bot } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { ScrollArea } from '@/components/ui/scroll-area';
import { ChatMessage } from '@/components/chat-message';
import { handleWasteQuery, type AIResponse } from '@/app/actions';
import { Card } from '@/components/ui/card';

export interface Message {
  id: string;
  role: 'user' | 'ai';
  text: string;
  category?: string;
}

const initialMessages: Message[] = [
  {
    id: 'init',
    role: 'ai',
    text: "안녕하세요! 저는 EcoSort AI입니다. 오늘 분리수거에 대해 무엇을 도와드릴까요? 버리려는 품목을 알려주세요.",
    category: 'greeting'
  }
];

export function ChatInterface() {
  const [messages, setMessages] = useState<Message[]>(initialMessages);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const scrollAreaRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    // 메시지 목록이 변경될 때마다 스크롤을 맨 아래로 이동
    const viewport = scrollAreaRef.current?.children[1] as HTMLElement;
    if (viewport) {
      viewport.scrollTop = viewport.scrollHeight;
    }
  }, [messages, isLoading]);

  const handleSubmit = async (e: FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    if (!input.trim() || isLoading) return;

    const userMessage: Message = { id: Date.now().toString(), role: 'user', text: input };
    const newMessages = [...messages, userMessage];
    setMessages(newMessages);
    setInput('');
    setIsLoading(true);

    try {
      // AI에 전달할 대화 기록 생성 (인사 메시지 제외)
      const historyForAI = newMessages.filter(m => m.category !== 'greeting').map(({ role, text }) => ({ role, text }));
      const aiResponse: AIResponse = await handleWasteQuery(historyForAI);
      
      const aiMessage: Message = {
        id: (Date.now() + 1).toString(),
        role: 'ai',
        text: aiResponse.text,
        category: aiResponse.category,
      };
      setMessages(prev => [...prev, aiMessage]);
    } catch (error) {
      const errorMessage: Message = {
        id: (Date.now() + 1).toString(),
        role: 'ai',
        text: "죄송합니다. 문제가 발생했습니다. 다시 시도해 주세요.",
        category: 'error',
      };
      setMessages(prev => [...prev, errorMessage]);
    } finally {
      setIsLoading(false);
    }
  };

  // ... (JSX 렌더링 부분 생략)
}
```

## 4. 메시지 렌더링 컴포넌트 - `src/components/chat-message.tsx`

사용자와 AI의 메시지를 화면에 표시하는 컴포넌트입니다. AI의 답변에 포함된 줄 바꿈(`\n`)과 마크다운(`**bold**`)을 HTML로 올바르게 렌더링하는 로직이 포함되어 있습니다.

```typescript
import { Bot, User } from 'lucide-react';
import { cn } from '@/lib/utils';
import type { Message } from './chat-interface';
import { Badge } from '@/components/ui/badge';
import { WasteCategoryIcon } from '@/components/icons/waste-category-icon';

export function ChatMessage({ message }: { message: Message }) {
  const isUser = message.role === 'user';
  const isSpecial = message.category === 'greeting' || message.category === 'error';

  const formatText = (text: string) => {
    // **text** -> <b>text</b> 로 변환하고,
    // 숫자 목록 앞에서는 강제로 줄바꿈(<br/>) 처리합니다.
    const processedText = text
      .replace(/\*\*(.*?)\*\*/g, '<b>$1</b>')
      .replace(/(\n)?(\d+\.)/g, '<br/>$2');
    
    // 맨 앞에 불필요한 <br/>이 있으면 제거합니다.
    const finalHtml = processedText.startsWith('<br/>') ? processedText.substring(5) : processedText;
      
    // HTML을 직접 렌더링합니다.
    return <div dangerouslySetInnerHTML={{ __html: finalHtml }} />;
  };

  return (
    <div
      className={cn(
        'flex items-start gap-4',
        isUser ? 'justify-end' : 'justify-start'
      )}
    >
      {!isUser && (
        <div className="flex-shrink-0 w-10 h-10 rounded-full bg-primary/20 flex items-center justify-center">
          <Bot className="w-6 h-6 text-primary" />
        </div>
      )}

      <div
        className={cn(
          'max-w-xl rounded-2xl px-5 py-4 shadow-md',
          isUser
            ? 'bg-primary text-primary-foreground rounded-br-lg'
            : 'bg-card text-card-foreground rounded-bl-lg'
        )}
      >
        {message.category && !isSpecial && (
          <div className="flex items-center gap-3 mb-3">
            <WasteCategoryIcon category={message.category} className="w-7 h-7 text-primary" />
            <Badge variant="secondary" className="text-base font-semibold">{message.category}</Badge>
          </div>
        )}
        <div className="text-sm space-y-2 break-words">
          {formatText(message.text)}
        </div>
      </div>

      {isUser && (
        <div className="flex-shrink-0 w-10 h-10 rounded-full bg-accent/20 flex items-center justify-center">
          <User className="w-6 h-6 text-accent" />
        </div>
      )}
    </div>
  );
}
```
