# Canonical Form Modal Example

Use this when a Zustand-controlled modal contains inputs, Zod validation, and an API request. Adapt the mutation API and error presentation to the repository.

## Trigger With Context

The trigger passes only contextual data needed by the modal. A button nested anywhere inside a form must declare `type="button"`.

```tsx
const onOpen = useModal(state => state.onOpen)

const handleReportComment = () => {
    onOpen('reportComment', {
        reviewId: String(review.id),
    })
}

return (
    <button type="button" onClick={handleReportComment}>
        Report
    </button>
)
```

Add the corresponding payload field to the modal store:

```ts
export interface ModalData {
    reviewId?: string
    replyId?: string
}
```

Prefer a discriminated payload map when stronger per-modal typing is practical.

## Schema And Form

```tsx
import { useEffect, useState } from 'react'
import { zodResolver } from '@hookform/resolvers/zod'
import { useForm } from 'react-hook-form'
import { z } from 'zod'
import {
    Dialog,
    DialogContent,
    DialogHeader,
    DialogTitle,
} from '@/components/ui/dialog'
import {
    Form,
    FormControl,
    FormField,
    FormItem,
    FormLabel,
    FormMessage,
} from '@/components/ui/form'
import { Button } from '@/components/ui/button'
import { Checkbox } from '@/components/ui/checkbox'
import { Input } from '@/components/ui/input'
import { Textarea } from '@/components/ui/textarea'
import { useModal } from '@/hooks/use-modal-store'
import { createReportComment } from '@/services/api'

const reportSchema = z
    .object({
        name: z.string().trim().min(1, 'Name is required'),
        email: z
            .string()
            .email('Enter a valid email')
            .or(z.literal('')),
        reason: z.object({
            inaccurateInfo: z.boolean(),
            violation: z.boolean(),
            suspiciousActivity: z.boolean(),
            other: z.boolean(),
        }),
        details: z.string().trim(),
        additionalInfo: z.string().trim(),
        confirm: z
            .boolean()
            .refine(value => value, 'Please confirm'),
    })
    .superRefine((values, context) => {
        const hasReason = Object.values(values.reason).some(Boolean)

        if (!hasReason) {
            context.addIssue({
                code: z.ZodIssueCode.custom,
                message: 'Select at least one reason',
                path: ['reason'],
            })
        }

        if (values.reason.other && !values.details) {
            context.addIssue({
                code: z.ZodIssueCode.custom,
                message: 'Please specify the other reason',
                path: ['details'],
            })
        }
    })

type ReportFormValues = z.infer<typeof reportSchema>

const defaultValues: ReportFormValues = {
    name: '',
    email: '',
    reason: {
        inaccurateInfo: false,
        violation: false,
        suspiciousActivity: false,
        other: false,
    },
    details: '',
    additionalInfo: '',
    confirm: false,
}
```

## Modal Lifecycle And Submission

```tsx
const ReportCommentModal = () => {
    const isOpen = useModal(state => state.isOpen)
    const type = useModal(state => state.type)
    const data = useModal(state => state.data)
    const onClose = useModal(state => state.onClose)
    const onOpen = useModal(state => state.onOpen)
    const [submitError, setSubmitError] = useState<string | null>(null)

    const isModalOpen = isOpen && type === 'reportComment'

    const form = useForm<ReportFormValues>({
        resolver: zodResolver(reportSchema),
        defaultValues,
    })

    useEffect(() => {
        if (!isModalOpen) {
            form.reset(defaultValues)
            setSubmitError(null)
        }
    }, [form, isModalOpen])

    const onSubmit = async (values: ReportFormValues) => {
        const reviewId = data.reviewId

        if (!reviewId) {
            setSubmitError('The review could not be identified.')
            return
        }

        setSubmitError(null)

        try {
            await createReportComment({
                ...values,
                email: values.email || undefined,
                review_id: reviewId,
                reply_id: data.replyId,
            })

            form.reset(defaultValues)
            onOpen('reportSubmitted')
        } catch (error) {
            setSubmitError(
                error instanceof Error
                    ? error.message
                    : 'The report could not be submitted.'
            )
        }
    }

    return (
        <Dialog
            open={isModalOpen}
            onOpenChange={open => {
                if (!open) onClose()
            }}
        >
            <DialogContent
                className="max-h-[90vh] w-[calc(100vw-2rem)] max-w-2xl overflow-y-auto"
                aria-describedby={undefined}
            >
                <DialogHeader>
                    <DialogTitle className="text-center">
                        Report comment
                    </DialogTitle>
                </DialogHeader>

                <Form {...form}>
                    <form
                        className="space-y-4"
                        onSubmit={form.handleSubmit(onSubmit)}
                        noValidate
                    >
                        <FormField
                            control={form.control}
                            name="name"
                            render={({ field }) => (
                                <FormItem>
                                    <FormLabel>Name</FormLabel>
                                    <FormControl>
                                        <Input
                                            {...field}
                                            autoComplete="name"
                                            placeholder="First and last name"
                                        />
                                    </FormControl>
                                    <FormMessage />
                                </FormItem>
                            )}
                        />

                        <FormField
                            control={form.control}
                            name="reason.other"
                            render={({ field }) => (
                                <FormItem className="flex items-start gap-2">
                                    <FormControl>
                                        <Checkbox
                                            checked={field.value}
                                            onCheckedChange={checked =>
                                                field.onChange(checked === true)
                                            }
                                        />
                                    </FormControl>
                                    <FormLabel>Other</FormLabel>
                                </FormItem>
                            )}
                        />

                        {/* Render the remaining reason checkboxes the same way. */}

                        <FormField
                            control={form.control}
                            name="details"
                            render={({ field }) => (
                                <FormItem>
                                    <FormLabel>Details</FormLabel>
                                    <FormControl>
                                        <Textarea {...field} />
                                    </FormControl>
                                    <FormMessage />
                                </FormItem>
                            )}
                        />

                        {form.formState.errors.reason?.message && (
                            <p role="alert" className="text-sm text-destructive">
                                {form.formState.errors.reason.message}
                            </p>
                        )}

                        <FormField
                            control={form.control}
                            name="confirm"
                            render={({ field }) => (
                                <FormItem>
                                    <div className="flex items-start gap-2">
                                        <FormControl>
                                            <Checkbox
                                                checked={field.value}
                                                onCheckedChange={checked =>
                                                    field.onChange(
                                                        checked === true
                                                    )
                                                }
                                            />
                                        </FormControl>
                                        <FormLabel>
                                            I confirm this report is accurate
                                        </FormLabel>
                                    </div>
                                    <FormMessage />
                                </FormItem>
                            )}
                        />

                        {submitError && (
                            <p role="alert" className="text-sm text-destructive">
                                {submitError}
                            </p>
                        )}

                        <Button
                            type="submit"
                            disabled={form.formState.isSubmitting}
                        >
                            {form.formState.isSubmitting
                                ? 'Submitting...'
                                : 'Submit'}
                        </Button>
                    </form>
                </Form>
            </DialogContent>
        </Dialog>
    )
}
```

When the repository uses TanStack Query or another mutation hook, use its pending state and success/error callbacks instead of adding duplicate local request state. Opening the success modal directly replaces the active modal state; use animation-completion coordination only when the UI library genuinely requires it.

## Required Tests

- The trigger passes the correct record identifier and does not submit an ancestor form.
- Invalid fields and conditional fields show messages.
- Checkbox values stored in the form are booleans.
- Repeated clicks cannot send duplicate requests.
- Success transitions to the intended modal and clears old payload data.
- Failure leaves the form open with its values intact and shows an error.
- Closing and reopening restores the intended default values.
