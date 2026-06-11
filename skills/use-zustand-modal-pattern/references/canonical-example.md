# Canonical TypeScript Modal Example

Use this as a structural reference, then adapt names, paths, styling, and payload fields to the repository.

## Zustand Store

```ts
import { create } from 'zustand'

export type ModalType =
    | 'login'
    | 'signup'
    | 'forgot-password'
    | 'photoGallery'
    | 'calendarMeeting'
    | 'listingFilterPanel'

export interface ModalData {
    images?: Array<string | File>
    initialImageIndex?: number
    listingId?: string
}

interface ModalStore {
    type: ModalType | null
    isOpen: boolean
    data: ModalData
    onOpen: (type: ModalType, data?: ModalData) => void
    onClose: () => void
}

export const useModal = create<ModalStore>(set => ({
    type: null,
    isOpen: false,
    data: {},
    onOpen: (type, data = {}) => set({ type, data, isOpen: true }),
    onClose: () => set({ type: null, data: {}, isOpen: false }),
}))
```

Do not include a `'none'` modal type when `null` already represents no active modal.

## Trigger Component

```tsx
import { Image } from 'lucide-react'
import PhotoGalleryModal from './PhotoGalleryModal'
import { useModal } from '@/store/modalStore'

interface ImageCarouselProps {
    images: Array<string | File>
    matterport?: string
}

const ImageCarousel = ({ images, matterport }: ImageCarouselProps) => {
    const onOpen = useModal(state => state.onOpen)

    const handleImagePreview = (event: React.MouseEvent<HTMLButtonElement>) => {
        event.stopPropagation()
        onOpen('photoGallery', { images, initialImageIndex: 0 })
    }

    if (images.length === 0) return null

    return (
        <div className="flex w-full flex-col items-center justify-center p-4">
            {/* Render the repository's carousel here using the images prop. */}

            <div className="mt-4 flex justify-center lg:self-end">
                <button
                    type="button"
                    onClick={handleImagePreview}
                    className="flex items-center rounded-3xl border-2 border-[#0045F1] px-4 py-2 text-base text-[#0045F1]"
                >
                    <Image aria-hidden="true" className="mr-2 size-6" />
                    See all photos
                </button>

                {matterport && (
                    <a
                        href={matterport}
                        target="_blank"
                        rel="noopener noreferrer"
                        className="ml-4 flex items-center rounded-3xl bg-[#0045F1] px-4 py-2 text-base text-white"
                    >
                        Explore 3D Tour
                    </a>
                )}
            </div>

            <PhotoGalleryModal />
        </div>
    )
}

export default ImageCarousel
```

If multiple trigger components can render at once, mount `PhotoGalleryModal` once in a shared modal provider instead.

## Dedicated Modal Component

```tsx
import { useEffect, useState } from 'react'
import { ArrowLeft } from 'lucide-react'
import {
    Dialog,
    DialogClose,
    DialogContent,
    DialogHeader,
    DialogTitle,
} from '@/components/ui/dialog'
import { useModal } from '@/store/modalStore'

interface GalleryImageProps {
    image: string | File
    index: number
}

const GalleryImage = ({ image, index }: GalleryImageProps) => {
    const [src, setSrc] = useState(typeof image === 'string' ? image : '')

    useEffect(() => {
        if (typeof image === 'string') {
            setSrc(image)
            return
        }

        const objectUrl = URL.createObjectURL(image)
        setSrc(objectUrl)

        return () => URL.revokeObjectURL(objectUrl)
    }, [image])

    if (!src) return null

    return (
        <img
            src={src}
            alt={`Property photo ${index + 1}`}
            className="size-full object-cover"
        />
    )
}

const PhotoGalleryModal = () => {
    const isOpen = useModal(state => state.isOpen)
    const type = useModal(state => state.type)
    const data = useModal(state => state.data)
    const onClose = useModal(state => state.onClose)

    const isModalOpen = isOpen && type === 'photoGallery'
    const images = data.images ?? []

    return (
        <Dialog
            open={isModalOpen}
            onOpenChange={open => {
                if (!open) onClose()
            }}
        >
            <DialogContent
                className="max-h-[80vh] w-[calc(100vw-2rem)] max-w-4xl"
                aria-describedby={undefined}
            >
                <DialogHeader className="flex-row items-center">
                    <DialogClose asChild>
                        <button type="button" aria-label="Close photo gallery">
                            <ArrowLeft aria-hidden="true" />
                        </button>
                    </DialogClose>
                    <DialogTitle className="flex-grow text-center text-2xl">
                        All Photos
                    </DialogTitle>
                </DialogHeader>

                <div className="grid grid-cols-1 gap-4 overflow-y-auto px-4 sm:grid-cols-2">
                    {images.map((image, index) => (
                        <div
                            key={`${typeof image === 'string' ? image : image.name}-${index}`}
                            className="relative aspect-square overflow-hidden"
                        >
                            <GalleryImage image={image} index={index} />
                        </div>
                    ))}
                </div>
            </DialogContent>
        </Dialog>
    )
}

export default PhotoGalleryModal
```
