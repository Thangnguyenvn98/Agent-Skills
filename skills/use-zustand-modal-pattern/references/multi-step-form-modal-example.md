# Canonical Multi-Step Form Modal Example

Use this when a modal coordinates two or more steps, such as selecting a service card, entering details, confirming a request, completing checkout, or collecting any staged input. The pattern is a reusable wizard: a manager controls modal visibility, current step, shared draft data, and transitions.

Keep the example generic. Rename the modal type, step names, draft fields, schemas, and API calls to match the feature being built.

## Manager Responsibilities

The modal manager should own orchestration:

- Read `isOpen`, `type`, and `onClose` from the modal store.
- Track the active step as a narrow union.
- Reset the active step and shared draft state on close.
- Render one step component at a time.
- Pass intent callbacks to steps, such as `onSelect`, `onNext`, `onBack`, and `onComplete`.
- Keep `DialogTitle` and `DialogDescription` inside `DialogContent`.

```tsx
import { useState } from 'react'
import {
    Dialog,
    DialogContent,
    DialogDescription,
    DialogTitle,
} from '@/components/ui/dialog'
import { useModal } from '@/hooks/use-modal-store'
import { useServiceRequestDraftStore } from '@/hooks/use-service-request-draft-store'
import ChooseServiceStep from './ChooseServiceStep'
import ServiceDetailsStep from './ServiceDetailsStep'

type ServiceRequestStep = 'choose-service' | 'details'

const ServiceRequestModal = () => {
    const isOpen = useModal(state => state.isOpen)
    const type = useModal(state => state.type)
    const onClose = useModal(state => state.onClose)
    const resetDraft = useServiceRequestDraftStore(state => state.resetDraft)

    const [step, setStep] = useState<ServiceRequestStep>('choose-service')
    const isModalOpen = isOpen && type === 'serviceRequest'

    const closeModal = () => {
        resetDraft()
        setStep('choose-service')
        onClose()
    }

    return (
        <Dialog
            open={isModalOpen}
            onOpenChange={open => {
                if (!open) closeModal()
            }}
        >
            <DialogContent
                aria-describedby="service-request-description"
                className="max-h-[90vh] w-[calc(100vw-2rem)] max-w-3xl overflow-y-auto"
            >
                <DialogTitle>Request a service</DialogTitle>
                <DialogDescription id="service-request-description">
                    Select a service, then provide the details needed to submit
                    the request.
                </DialogDescription>

                {step === 'choose-service' && (
                    <ChooseServiceStep onNext={() => setStep('details')} />
                )}

                {step === 'details' && (
                    <ServiceDetailsStep
                        onBack={() => setStep('choose-service')}
                        onComplete={closeModal}
                    />
                )}
            </DialogContent>
        </Dialog>
    )
}

export default ServiceRequestModal
```

For a three-step flow, add another named step such as `'review'` and pass `onNext={() => setStep('review')}` from the details step. Do not add extra steps unless they clarify the user journey.

## Shared Draft Store

Use a typed draft store or parent state for values that must survive across steps. Keep this separate from the modal store:

- The modal store answers "which modal is open?"
- The draft store answers "what has the user selected or entered so far?"

```ts
import { create } from 'zustand'

interface ServiceRequestDraft {
    serviceId?: string
    serviceName?: string
    notes?: string
    preferredDate?: string
}

interface ServiceRequestDraftStore {
    draft: ServiceRequestDraft
    updateDraft: (data: Partial<ServiceRequestDraft>) => void
    resetDraft: () => void
}

export const useServiceRequestDraftStore =
    create<ServiceRequestDraftStore>(set => ({
        draft: {},
        updateDraft: data =>
            set(state => ({
                draft: {
                    ...state.draft,
                    ...data,
                },
            })),
        resetDraft: () => set({ draft: {} }),
    }))
```

## Card Selection Step

For a first step where the user selects a card, the step may not need React Hook Form. Use accessible buttons, store the selected value in the draft, and disable `Next` until the user selects an option.

```tsx
import { Button } from '@/components/ui/button'
import { cn } from '@/lib/utils'
import { useServiceRequestDraftStore } from '@/hooks/use-service-request-draft-store'

const services = [
    { id: 'cleaning', label: 'Cleaning', description: 'Home cleaning help' },
    { id: 'moving', label: 'Moving', description: 'Moving and lifting help' },
    { id: 'repair', label: 'Repair', description: 'Small home repairs' },
]

interface ChooseServiceStepProps {
    onNext: () => void
}

const ChooseServiceStep = ({ onNext }: ChooseServiceStepProps) => {
    const draft = useServiceRequestDraftStore(state => state.draft)
    const updateDraft = useServiceRequestDraftStore(state => state.updateDraft)

    const selectedServiceId = draft.serviceId

    return (
        <div className="space-y-6">
            <div className="grid gap-4 sm:grid-cols-3">
                {services.map(service => {
                    const selected = selectedServiceId === service.id

                    return (
                        <button
                            key={service.id}
                            type="button"
                            aria-pressed={selected}
                            onClick={() =>
                                updateDraft({
                                    serviceId: service.id,
                                    serviceName: service.label,
                                })
                            }
                            className={cn(
                                'rounded-xl border p-4 text-left',
                                selected && 'border-primary bg-primary/10'
                            )}
                        >
                            <span className="block font-semibold">
                                {service.label}
                            </span>
                            <span className="block text-sm text-muted-foreground">
                                {service.description}
                            </span>
                        </button>
                    )
                })}
            </div>

            <div className="flex justify-end">
                <Button
                    type="button"
                    disabled={!selectedServiceId}
                    onClick={onNext}
                >
                    Next
                </Button>
            </div>
        </div>
    )
}

export default ChooseServiceStep
```

If changing the selected card invalidates later answers, clear those dependent fields when updating the draft.

## Form Step

Each form step should own its validation when schemas differ. On submit, merge validated values with the existing draft and submit or continue.

```tsx
import { zodResolver } from '@hookform/resolvers/zod'
import { useForm } from 'react-hook-form'
import { z } from 'zod'
import {
    Form,
    FormControl,
    FormField,
    FormItem,
    FormMessage,
} from '@/components/ui/form'
import { Button } from '@/components/ui/button'
import { Textarea } from '@/components/ui/textarea'
import { Input } from '@/components/ui/input'
import { useServiceRequestDraftStore } from '@/hooks/use-service-request-draft-store'
import { useCreateServiceRequestMutation } from '@/services/mutations'

const detailsSchema = z.object({
    notes: z.string().min(10, 'Add a few more details'),
    preferredDate: z.string().min(1, 'Choose a preferred date'),
})

type DetailsValues = z.infer<typeof detailsSchema>

interface ServiceDetailsStepProps {
    onBack: () => void
    onComplete: () => void
}

const ServiceDetailsStep = ({
    onBack,
    onComplete,
}: ServiceDetailsStepProps) => {
    const draft = useServiceRequestDraftStore(state => state.draft)
    const updateDraft = useServiceRequestDraftStore(state => state.updateDraft)
    const createRequest = useCreateServiceRequestMutation()

    const form = useForm<DetailsValues>({
        resolver: zodResolver(detailsSchema),
        defaultValues: {
            notes: draft.notes ?? '',
            preferredDate: draft.preferredDate ?? '',
        },
    })

    const onSubmit = (values: DetailsValues) => {
        if (!draft.serviceId) {
            onBack()
            return
        }

        const payload = {
            ...draft,
            ...values,
            serviceId: draft.serviceId,
        }

        createRequest.mutate(payload, {
            onSuccess: () => {
                updateDraft(values)
                onComplete()
            },
            onError: error => {
                form.setError('root', {
                    message:
                        error instanceof Error
                            ? error.message
                            : 'The request could not be submitted.',
                })
            },
        })
    }

    return (
        <Form {...form}>
            <form className="space-y-4" onSubmit={form.handleSubmit(onSubmit)}>
                {form.formState.errors.root?.message && (
                    <p role="alert" className="text-sm text-destructive">
                        {form.formState.errors.root.message}
                    </p>
                )}

                <FormField
                    control={form.control}
                    name="notes"
                    render={({ field }) => (
                        <FormItem>
                            <FormControl>
                                <Textarea
                                    {...field}
                                    placeholder="Tell us what you need"
                                />
                            </FormControl>
                            <FormMessage />
                        </FormItem>
                    )}
                />

                <FormField
                    control={form.control}
                    name="preferredDate"
                    render={({ field }) => (
                        <FormItem>
                            <FormControl>
                                <Input {...field} type="date" />
                            </FormControl>
                            <FormMessage />
                        </FormItem>
                    )}
                />

                <div className="flex justify-end gap-4">
                    <Button type="button" variant="ghost" onClick={onBack}>
                        Back
                    </Button>
                    <Button
                        type="submit"
                        disabled={createRequest.isPending}
                    >
                        {createRequest.isPending ? 'Submitting...' : 'Submit'}
                    </Button>
                </div>
            </form>
        </Form>
    )
}

export default ServiceDetailsStep
```

The final submit composes the payload synchronously from the draft and current form values. Avoid a "set draft, set flag, submit in useEffect" sequence because it can submit stale data.

## Branching From A Selection

When the first step controls later fields:

- Store the selected value in the draft, such as `serviceId`.
- Choose schemas from a small function such as `getDetailsSchema(serviceId)`.
- Keep branch-specific defaults complete enough for controlled inputs.
- If the user changes the selection, clear fields that no longer apply.
- Guard later steps from missing required selection data by sending the user back to the selection step.

```ts
const getDetailsSchema = (serviceId: string | undefined) => {
    if (serviceId === 'moving') return movingDetailsSchema
    if (serviceId === 'repair') return repairDetailsSchema
    if (serviceId === 'cleaning') return cleaningDetailsSchema
    return null
}
```

Avoid silently falling back to an empty schema when required branch data is missing. Redirect to the step where the missing data should be collected.

## Checklist

- Modal manager derives open state from Zustand and active modal type.
- Close resets modal store, active step, form draft, and transient UI flags.
- Child steps own local validation and only report validated data upward.
- Navigation callbacks express intent rather than exposing raw setters.
- Card choices are buttons with `type="button"`, `aria-pressed`, and visible selected state.
- Checkboxes normalize Radix values with `checked === true`.
- Final submit uses a composed payload and server errors keep the user on the current step.
- Changing an earlier card or branch clears dependent later data when needed.
- Titles and descriptions are accessible.
- Tests cover step navigation, close/reopen reset, branch changes, final payload, success, and server failure.
